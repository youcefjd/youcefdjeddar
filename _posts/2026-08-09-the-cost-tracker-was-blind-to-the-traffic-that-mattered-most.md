---
layout: post
title: "Day 4: The Cost Tracker Was Blind to the Traffic That Mattered Most"
date: 2026-08-09
description: "Buffered requests were trivial to bill. Streaming ones never hit the recorder at all — and streaming is most of the traffic. Closing the gap also surfaced a hardcoded token cap and turned TTFT into a metric the platform reports on its own."
---

[Day 3]({% link _posts/2026-07-31-the-router-the-scheduler-and-the-hostile-test.md %}) shipped a router and a scheduler — something that decides which model answers and when. Once that was running, the next question was obvious: who's actually driving the bill?

So I built cost attribution. Tokens times price, per principal, live on the endpoint. Every request, I knew who asked, what it cost, and how that added up over time.

It worked. I trusted it.

Then I went looking at what it was actually recording, and the streaming path had never touched it.

---

## Why This Number Is the Point

The rate card is fixed — I don't set what Claude or a local model costs per token. The thing I actually control is knowing where the spend goes: which principal, which workload, which requests are burning frontier tokens on something a local model could've handled.

That's a governance problem before it's an accounting one. AI spend is one of the fastest-growing line items on any team's bill, and most places have no idea which request, agent, or person is driving it. "We spent $X this month" is a useless number. "Agent Y drove 40% of frontier spend running the same prompt in a retry loop" is the one that changes what you do next.

This also can't live in the model-serving layer. vLLM doesn't know who's asking or what the request is worth — same blindness [Day 3]({% link _posts/2026-07-31-the-router-the-scheduler-and-the-hostile-test.md %}) found in the scheduler. Attribution, like priority, has to live in the layer that actually knows who the caller is.

---

## The Easy Version, and the Hole

Buffered requests were trivial: await the full response, read the usage block off it, record it. One moment, all the data sitting right there.

Streaming doesn't have that moment. There's no single point where the whole response exists — just a sequence of fragments arriving over time, and a request lifecycle with three distinct points (first token, every chunk after, stream end) instead of one.

The hole wasn't a bug in code that ran. It was a gap in what got wired up — the streaming path had simply never been connected to the recorder. Nothing threw. Nothing logged an error. The dashboard kept updating, just from buffered requests only, and in production chat traffic is almost entirely streaming. So the total-spend number — the one the whole tracker exists to get right — was quietly built on the minority of the traffic.

Here's the part I can't clean up with a number: I don't know how much it undercounted by. Not because I didn't check — because the gap couldn't have left a number behind. Recording spend was exactly the thing that wasn't happening on that path, which means there's no log to go back and total up. Asking "how much did we miss" assumes a record that, for that traffic, never existed.

That's a worse answer than a percentage, and it's the actual point. A gap you can size after the fact was never that dangerous — you find it, you quantify it, you move on. This one didn't leave that option. Some real, unrecoverable amount of frontier spend went unattributed for as long as the streaming path was disconnected, and the honest number is "unknown," not a plausible-looking stand-in for it.

That's the dangerous kind of gap. A crash gets fixed by lunch. A dashboard that's confidently short doesn't ask to be checked — and when you finally do check, it can't even tell you how wrong it was.

---

## Closing It

Two changes, both in the streaming path itself.

**Move the hook inside the generator, record in `finally`.**

```python
async def stream_and_record(request, backend):
    tokens_in = count_prompt_tokens(request)
    tokens_out = 0
    try:
        async for chunk in backend.stream(request):
            tokens_out += count_delta_tokens(chunk)
            yield chunk
    finally:
        cost_recorder.record(
            principal=request.principal,
            model=request.model,
            tokens_in=tokens_in,
            tokens_out=tokens_out,
        )
```

`finally` matters for a specific reason: if the client disconnects mid-stream, the request still gets recorded. I still paid for the compute that already ran, whether or not anyone was left to read the rest of the tokens.

**Own the accuracy gap instead of hiding it.**

Claude sends exact usage in a `message_delta` event right before the stream closes — the field was there the whole time, I just wasn't reading it. vLLM's OpenAI-compatible server can send usage too, but only if the request sets `stream_options: {"include_usage": true}`; the router wasn't setting it. Without that flag, local falls back to counting content chunks as they arrive — close, but not exact.

So the tracker is now honest about a real limitation instead of quietly pretending both numbers mean the same thing: exact for frontier, approximate for local.

I'm not going to put a fixed error rate on that approximation, because there isn't one to quote. Chunk-counting drifts with however the tokenizer happened to segment that particular response — the "Ele" then "ven" split from Day 3 is the reminder that a fragment isn't reliably one token. Some completions it'll be close to exact; others it'll drift more, and the drift isn't a constant I can subtract out. "Approximate" is the honest word here, not a percentage I'd have to make up to sound more precise than the method actually is.

**The bonus find.** Reading that `message_delta` event to get usage also surfaced `stop_reason` — a field I'd been dropping on the floor along with the token counts.

Both `call_frontier` (buffered) and `stream_frontier` (streaming) build an `anthropic_req` dict with `"max_tokens": 2048` hardcoded in it. It's there because Anthropic's API requires `max_tokens` on every request — unlike the OpenAI-style endpoint, where it's optional. Back when I first wired the frontier path, 2048 went in as a placeholder just to satisfy the field and move on. It's been sitting there since, flagged as debt, never revisited.

That's a hard ceiling on Claude's output. A long proof, a detailed explanation, a big code block — anything that wants more than 2048 tokens gets cut off mid-sentence at exactly that number, and comes back with `stop_reason: "max_tokens"` instead of `"end_turn"`. Before today, nothing was reading `stop_reason` at all, so a clip like that would land as a normal-looking, if abruptly short, answer. No signal it was cut rather than finished.

I'd already learned this lesson once — Day 2's `max_tokens: 2048` cap on the eval harness is what quietly turned a 95% accuracy into a fake 71%. Same number, same failure mode, dormant in a different part of the same codebase, for the same reason: a cap set once under time pressure and never revisited. The only difference is I found this one before it cost me a wrong number in production, because the instrumentation I just built to fix billing happens to also make truncation visible — a clip now shows up as `stop_reason: "max_tokens"` in the stats. One fix closed a billing gap and turned a second, dormant bug from invisible to visible.

I haven't raised the cap yet. The fix reads trivial — bump it to 4096 or 8192, or make it configurable per request — but it isn't a pure win, and I want to say why before closing this out.

---

## The Loop Closes: TTFT Becomes a Platform Number

[Day 1]({% link _posts/2026-07-15-standing-up-the-inference-seam.md %}) found the number that's been steering this whole project: the same GPU, same instant, gives a human 14x worse latency and an agent fleet 42x more throughput, depending on who's asking (16.7ms → 230ms avg TTFT, 95 → 4047 tok/s aggregate, concurrency 1 to 64). That split is the reason a scheduler exists at all.

[Day 2]({% link _posts/2026-07-17-the-quantization-frontier-what-it-actually-costs.md %}) ended on a regret: `vllm`'s `/metrics` endpoint had queue depth sitting there the whole time, and I'd spent a day inferring it from latency spread instead of just asking for it.

Both of those close here. TTFT is now recorded per request, split out from total latency, and reported live instead of reconstructed after the fact from a synthetic sweep. TTFT is what a human waiting on a chat reply actually feels. Total latency is the throughput story an agent fleet cares about. A platform that reports both, live, per request, is finally speaking the same language its own scheduler was built in — the number that justified building the thing is now a number the thing reports on itself.

---

## The Pattern, Named

Three times now, on this project, the number that looked fine was the one that was wrong.

- **Truncation, disguised as a passing score.** A `max_tokens` cap of 2048 on a math eval quietly clipped long reasoning traces mid-answer. Reported accuracy: 71%. Real accuracy once the cap was fixed: 95%. Nothing errored — the harness just scored a truncated trace and moved on. Same number, same failure mode, same root cause — a cap set once under time pressure and never revisited — turned up dormant in the frontier path today. I didn't reuse 2048 on purpose. It's just what "I'll fix this later" defaults to, apparently.
- **A composite metric that folded two different things into one number.** "Tokens per second" dropped 37% under long context and read like decode had collapsed. Decode had actually only lost 7%; the other 30 points were prefill latency hiding inside a throughput number.
- **A cost tracker undercounting by never watching the traffic that mattered.** Same shape as the other two: nothing crashed, the number just wasn't counting what I assumed it was.

Errors that throw are the easy case — they announce themselves. The ones that quietly return a plausible, wrong value are the ones that cost you, because you act on them without checking. The fix isn't cleverness. It's asking, every time a number looks clean, what it isn't counting.

That question is the actual throughline of this series, more than the router or the scheduler or the quantization table. The skill was never building the metric. It's distrusting it until it's earned the trust.

---

## What's Next

The 2048 cap isn't a bug I get to just close. Raising it is close to free by itself — Claude bills for tokens actually generated, not the ceiling, so a higher cap doesn't cost anything on responses that were never going to hit it. But the ceiling is also the only thing standing between a pathological request and an unbounded frontier bill. Remove it and I've traded "some long answers clip" for "nothing stops a runaway response from running up real money." That's not a bug fix, it's a policy call — what's the right ceiling, and is it even the same number for every caller.

Which argues it shouldn't be a global constant at all. The honest fix is a per-principal budget: a human asking for a detailed writeup gets more room than an agent making a programmatic call that should never need more than a few hundred tokens. That's the same shape as the priority-and-quota argument from Day 3 — a single fixed number never fits everyone, and the fix is always "make it a policy, keyed on who's asking," not "pick a better fixed number." I haven't built that yet. For now the cap stays at 2048, tracked, visible in `stop_reason` if it fires, and not silently trusted the way it was yesterday.

Attribution also needs backfilling for the window it was blind to streaming. And the router-plus-scheduler stack still hasn't run under the same concurrency sweep Day 1 ran on raw vLLM — still the actual open item from Day 3. Cost attribution was supposed to be a quick add on top of a stable system. It turned into a reminder that "stable" and "fully instrumented" aren't the same claim.
