---
layout: post
title: "Day 4: Fixing a Streaming Cost-Tracking Gap"
date: 2026-08-09
description: "Buffered requests were easy to bill. Streaming ones never reached the recorder, and streaming is most of the traffic. Fixing that also exposed a hardcoded token cap and made TTFT a metric the platform reports itself."
---

[Day 3]({% link _posts/2026-07-31-the-router-the-scheduler-and-the-hostile-test.md %}) shipped a router and a scheduler, something that decides which model answers and when. Once that was running, the next question was who's actually driving the bill.

So I built cost attribution: tokens times price, per principal, live on the endpoint. For every request I knew who asked, what it cost, and how it added up over time. It seemed to work, so I trusted it, until I looked at what it was actually recording and found the streaming path had never been connected to it.

---

## Why This Number Is the Point

The rate card is fixed, since I don't set what Claude or a local model costs per token. What I can control is knowing where the spend goes: which principal, which workload, which requests are burning frontier tokens on something a local model could've handled.

That's a governance problem as much as an accounting one. AI spend is one of the fastest-growing line items on a lot of teams' bills, and most places don't know which request, agent or person is driving it. "We spent $X this month" doesn't tell you much. "Agent Y drove 40% of frontier spend running the same prompt in a retry loop" tells you what to fix.

This also can't live in the model-serving layer. vLLM doesn't know who's asking or what the request is worth, the same limitation [Day 3]({% link _posts/2026-07-31-the-router-the-scheduler-and-the-hostile-test.md %}) found in the scheduler. Like priority, attribution has to live in the layer that knows who the caller is.

---

## The Easy Version, and the Hole

Buffered requests were easy: wait for the full response, read the usage block, record it. All the data is there at one moment.

Streaming doesn't have that moment. The whole response never exists in one place. It arrives as fragments over time, and the request has three points that matter (first token, each chunk after that, end of stream) instead of one.

The gap wasn't a bug in code that ran. The streaming path had just never been connected to the recorder. Nothing threw and nothing logged an error. The dashboard kept updating from buffered requests only, while production chat traffic is almost entirely streaming. So the total-spend number, the main thing the tracker exists for, was based on the smaller share of the traffic.

I also can't tell you how much it undercounted by. That's not because I didn't look. Recording spend was exactly what wasn't happening on that path, so there's no log to total up afterwards.

That's worse than having a percentage. If you can size a gap after the fact, you find it, quantify it and move on. Here, some real amount of frontier spend went unattributed for as long as the streaming path was disconnected, and the honest answer is "unknown." I'd rather say that than make up a plausible number.

A crash usually gets fixed quickly because it's obvious. A dashboard that's quietly short doesn't prompt anyone to check it, and when you do check, it can't tell you how far off it was.

---

## Closing It

I made two changes, both in the streaming path.

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

`finally` matters because if the client disconnects mid-stream, the request still gets recorded. I still paid for the compute that ran, whether or not anyone stuck around to read the rest.

**Own the accuracy gap instead of hiding it.**

Claude sends exact usage in a `message_delta` event right before the stream closes. The field was there the whole time and I just wasn't reading it. vLLM's OpenAI-compatible server can send usage too, but only if the request sets `stream_options: {"include_usage": true}`; the router wasn't setting it. Without that flag, the local path falls back to counting content chunks as they arrive, which is close but not exact.

So the tracker now says plainly that frontier numbers are exact and local numbers are approximate, instead of treating them as the same thing.

I'm not putting an error rate on that approximation, because there isn't a fixed one. Chunk counting drifts depending on how the tokenizer split that particular response (the "Ele" then "ven" split from Day 3 shows a fragment isn't always one token). Some completions will be close to exact and others will drift more, and the drift isn't a constant I can subtract. "Approximate" is accurate. A percentage would be made up.

**A bonus find.** Reading that `message_delta` event for usage also exposed `stop_reason`, another field I'd been ignoring along with the token counts.

Both `call_frontier` (buffered) and `stream_frontier` (streaming) build an `anthropic_req` dict with `"max_tokens": 2048` hardcoded in it. It's there because Anthropic's API requires `max_tokens` on every request, unlike the OpenAI-style endpoint where it's optional. When I first wired up the frontier path, I put in 2048 as a placeholder to satisfy the field, flagged it as debt, and never came back to it.

That's a hard ceiling on Claude's output. A long proof, a detailed explanation or a big code block that needs more than 2048 tokens gets cut off mid-sentence and comes back with `stop_reason: "max_tokens"` instead of `"end_turn"`. Until today nothing read `stop_reason`, so a cut-off answer would just look like a normal, slightly short one.

I'd already been bitten by this once. On Day 2, a `max_tokens: 2048` cap on the eval harness turned 95% accuracy into a fake 71%. It's the same number and the same failure, sitting in a different part of the codebase for the same reason: a cap set once under time pressure and never revisited. This time I found it before it produced a wrong number in production, because the instrumentation I built for billing also makes truncation visible. A cut-off response now shows up as `stop_reason: "max_tokens"` in the stats, so fixing billing also exposed a second bug.

I haven't raised the cap yet. It looks like a trivial fix (bump it to 4096 or 8192, or make it configurable per request), but it isn't purely a win, and I'll explain why at the end.

---

## TTFT Becomes a Platform Metric

[Day 1]({% link _posts/2026-07-15-standing-up-the-inference-seam.md %}) found the number that's been driving this whole project: the same GPU at the same moment gives a human 14x worse latency and an agent fleet 42x more throughput, depending on who's asking (16.7ms → 230ms avg TTFT, 95 → 4047 tok/s aggregate, concurrency 1 to 64). That split is why the scheduler exists.

[Day 2]({% link _posts/2026-07-17-the-quantization-frontier-what-it-actually-costs.md %}) ended with me realizing `vllm`'s `/metrics` endpoint had queue depth available the whole time, and I'd spent a day inferring it from latency spread instead of just reading it.

Both of those get resolved here. TTFT is now recorded per request, separately from total latency, and reported live instead of reconstructed later from a synthetic sweep. TTFT is what a person waiting on a chat reply notices, and total latency is what an agent fleet cares about. Now the platform reports both live, per request, including the number that justified building the scheduler in the first place.

---

## The Pattern

Three times now on this project, the number that looked fine was the one that was wrong.

- **Truncation, disguised as a passing score.** A `max_tokens` cap of 2048 on a math eval quietly clipped long reasoning traces mid-answer. Reported accuracy was 71%, and real accuracy once the cap was fixed was 95%. Nothing errored, the harness just scored truncated traces and moved on. The same cap with the same root cause turned up in the frontier path today. I didn't pick 2048 twice on purpose. Apparently it's just my default for "I'll fix this later."
- **A composite metric that folded two different things into one number.** "Tokens per second" dropped 37% under long context and read like decode had collapsed. Decode had only lost 7%, and the other 30 points were prefill latency mixed into a throughput number.
- **A cost tracker that never saw most of the traffic.** Same as the other two: nothing crashed, the number just wasn't counting what I assumed it was.

Errors that throw are the easy ones, because you see them. The ones that quietly return a plausible but wrong value cost more, because you act on them without checking. The only defense I've found is to ask, whenever a number looks clean, what it might not be counting.

That question has turned out to be the main theme of this series, more than the router, the scheduler or the quantization table. Building a metric is the easy part. The work is not trusting it until you've checked it.

---

## What's Next

The 2048 cap isn't something I can just close. Raising it costs almost nothing by itself, because Claude bills for tokens actually generated, not the ceiling. But the ceiling is also the only thing between a pathological request and an unbounded frontier bill. Remove it and "some long answers get cut off" becomes "nothing stops a runaway response from spending real money." So it's a policy decision: what the right ceiling is, and whether it should be the same for every caller.

I don't think it should be a global constant at all. The better fix is a per-principal budget, where a person asking for a detailed writeup gets more room than an agent making a programmatic call that should never need more than a few hundred tokens. It's the same argument as priorities and quotas in Day 3: one fixed number doesn't fit everyone, so make it a policy based on who's asking. I haven't built that yet. For now the cap stays at 2048, but it's tracked and visible in `stop_reason` when it fires.

Attribution also needs backfilling for the period it missed streaming, if that's even possible. And the router-plus-scheduler stack still hasn't been run under the same concurrency sweep Day 1 ran on raw vLLM, which is still the open item from Day 3. Cost attribution was supposed to be a quick addition to a stable system, and it reminded me that a system can be stable without being fully instrumented.
