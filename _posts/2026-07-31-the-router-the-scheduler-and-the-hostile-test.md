---
layout: post
title: "Day 3: The Router, the Scheduler, and the Hostile Test That Almost Fooled Me"
date: 2026-07-31
description: "Why serving tokens fast isn't the same problem as serving them intelligently — a streaming translator, a scheduler that isn't Kafka, and a test I had to make adversarial before it told me the truth."
---

[Day 1]({% link _posts/2026-07-15-standing-up-the-inference-seam.md %}) found the number that's been steering everything since: the same GPU, at the same instant, gives a human 14x worse latency and an agent fleet 42x more throughput, depending entirely on who's asking. [Day 2]({% link _posts/2026-07-17-the-quantization-frontier-what-it-actually-costs.md %}) spent a full day inside FP8 and ended at a fork — push into FP4, benchmark SGLang, or stop and ship the table.

I took none of those forks. Day 1's finding was still sitting there unresolved: a GPU with no single correct configuration for two classes of user, and no mechanism yet that acts on that fact. Sharpening the quantization table further felt like polishing a number nobody downstream could use. So today pivoted to what Day 1 actually demanded — something that decides, per request, which model answers and when it runs. A router and a scheduler. Before either gets a line of code, I want to be honest about why both have to exist, because "add a router" is easy to say and easy to build for the wrong reasons.

---

## Why a Router: Not Every Request Deserves the Same Model, or the Same Effort

Most people building on LLMs make one API call to one model and move on. Fine, when you have one model, one kind of user, and no cost pressure. It stops being fine the moment any of those three changes — and in a system where humans and agents are both first-class users of the same GPU pool, all three change at once.

That's the system I'm building: a platform where humans and agents send requests to the same pool of models, and the infrastructure has to serve them intelligently. Not a chatbot — the control plane underneath a fleet of them. Model serving is the credibility floor, not the point. The interesting part is everything between a request arriving and a model answering it.

Once you have more than one model — a cheap fast one and an expensive smart one — every incoming request poses a question: which one should answer this? Sending "say hi in three words" to a frontier model is waste. Sending "prove this theorem" to a tiny model is failure. Something has to decide, per request, where it goes. That decider is the router. It sits in front of the models, speaks the same API they do so callers can't tell it's there, and makes three kinds of call:

**How hard should the model think?** Day 2 already gave me the price tag: turning thinking mode on costs 9.1x the tokens (2421 vs 265) for +5.0 points of accuracy on GSM8K. Worth it on a hard problem. Waste on "hi" — which Day 1 already caught spending 108 tokens deliberating over a greeting. The router gates thinking mode per request instead of leaving it on a blanket default.

**Which model answers?** I ran the same proof — a short number-theory argument — through both backends today. The small local model flailed its way to a correct-ish answer in 3036 tokens. The frontier model did it clean in 498. Six times the tokens for a worse result. Most requests don't need that; a fair share do. The router's job is telling which is which before either backend spends a token.

**And it hides all of this.** The two backends don't even speak the same wire format — vLLM is OpenAI-shaped, Anthropic isn't. The router translates both directions so that from outside there's one uniform door. Same door, different rooms, caller never knows which one they walked into.

A router turns "I have some models" into "I have a system that routes work to the right model." The difference between a pile of endpoints and a platform.

## Why a Scheduler: One GPU, Two Users Who Want Opposite Things

Here's the fact that forces a scheduler into existence, and Day 1 measured it before I had a name for it. Generating text has two phases with opposite characteristics. Time-to-first-token explodes under load — Day 1 clocked it at 14x worse, 16.7ms to 230ms average, going from concurrency 1 to 64. Inter-token latency barely moves — up only 34% over the same 64x range, because the weight read dominates and extra sequences ride along mostly free.

Now overlay who's waiting. A human feels every millisecond of that first-token wait — staring at a screen. An agent grinding through a batch job doesn't care if it starts 200ms later; it cares about total throughput. So the two want the GPU tuned in opposite directions, and you can't maximize both for the same request at the same time. Not a preference. It's the curve Day 1 already plotted.

Which means: when a human request and an agent request are both waiting, serving the human first costs the agent almost nothing and saves the human the exact thing they feel. Free value — but only if something is deciding whose request goes first. That something is the scheduler. It's the piece that knows a request belongs to a human versus an agent — the model-serving layer has no idea, to it every request is just tokens — and orders them accordingly.

## The Shape of It

Put together: a request arrives, the router decides which model and how hard, the scheduler decides when it runs relative to everyone else, the model serves it, the answer streams back uniform regardless of which backend produced it. The router is what answers. The scheduler is when. Model serving sits underneath both, doing the actual generation — excellent at mechanical efficiency, deliberately blind to who's asking and what it's worth. That blindness is exactly the gap the router and scheduler fill.

Three things got built today to close it. In order of how much each one surprised me.

---

## Making the Router Stream — and Why the Frontier Path Is Harder Than It Looks

Buffered responses are a lie of convenience. Real chat streams token by token. But adding streaming to a router that fronts two different backends isn't one feature — it's two, because the backends don't speak the same stream.

**Local (vLLM) is a pure pipe.** vLLM already emits OpenAI-format SSE chunks, so the router just relays them through. Not interesting, and it shouldn't be.

**Frontier (Claude) is a live translator.** Anthropic emits typed events — `message_start`, `content_block_delta`, `message_stop` — structurally different from OpenAI's `chat.completion.chunk`:

```
event: message_start
data: {"type":"message_start", ...}

event: content_block_delta
data: {"type":"content_block_delta","delta":{"text":"Euclid"}}

event: message_stop
data: {"type":"message_stop"}
```

against the shape the caller actually expects:

```
data: {"choices":[{"delta":{"content":"Euclid"}}]}
data: [DONE]
```

The router has to parse each Anthropic event as it arrives, pull the text out of `content_block_delta`, re-wrap it in OpenAI's chunk shape, and manufacture the `data: [DONE]` sentinel Claude never sends — Anthropic just closes the connection at `message_stop` and expects you to know that means done.

The discipline point that mattered here: before writing the translator, I fired a raw `curl` at the Anthropic endpoint and read the actual event stream, rather than trusting my memory of the API shape. Model APIs drift. Ground truth is one `curl` away, and it's cheaper to check than to debug a translator built on a remembered shape that's a few months stale.

The payoff: the caller sends `"model": "Qwen3-8B"` and gets back a streamed proof from Claude, in vLLM's wire format, and can't tell the difference from the outside. That invisibility is the router. Same door, different rooms.

One honest wart, left un-fixed on purpose: stream granularity differs by backend. vLLM emits subword fragments — "Ele" then "ven" — because that's how its tokenizer segments. Claude emits whole clauses at a time. The router passes through whatever each backend chooses rather than trying to normalize the chunking. Smoothing it would add latency for a cosmetic win nobody asked for.

---

## vLLM Already Has a Scheduler. So What Am I Building?

Here's the moment that reframed the rest of the day. The instant I decided to build a scheduler in front of vLLM, I discovered vLLM already has one. Continuous batching — the thing that made Day 1's 42x throughput number possible — is a scheduler. It decides which sequence gets a token on each step, packs the KV cache, preempts when memory runs tight. State of the art. I am not rebuilding it.

So what's left? vLLM's scheduler optimizes mechanical efficiency *within one model*. It structurally cannot do three things: know who a request belongs to — human or agent — see across backends (it has no idea Claude exists), or apply a fairness policy of its own beyond "keep the GPU full."

The division falls out cleanly: **vLLM enforces, the router decides.** vLLM ships a `priority` field it will order requests by — but it can't compute what priority a request deserves. That mapping, principal to priority, is the policy brain, and it has to live above vLLM. Real serving stacks are basically always two layers: an efficiency engine underneath, and a policy layer on top deciding what the engine should be efficient *about*. I'm not building something exotic. I'm building the standard outer layer that's supposed to sit there.

I didn't trust that vLLM would actually honor the priority field just because a flag exists for it. I probed it. Restarted the server with `--scheduling-policy priority`, sent one request with `priority: 0`, confirmed it was accepted without error. Accepted is not the same as acted on — a single request can't prove ordering, because there's nothing to order it against. That needs contention.

So I made contention. Fired six heavy agent requests to saturate the GPU, then sent one human request dead last, after all six agents were already queued:

| Request | Priority | Launch order | Finish time |
|---|---|---|---|
| human | 0 | last | 0.14s |
| agent (x6) | 10 | first | 11–16s |

The request that arrived last finished roughly 100x faster than the requests ahead of it, because priority let it cut the queue. Day 1's thesis — humans and agents want the GPU tuned in opposite directions — stopped being an argument and became a measurement. And per vLLM's own published benchmarks, priority scheduling costs close to zero aggregate throughput to run. Free lunch, and I confirmed it on my own hardware instead of taking the benchmark's word for it.

---

## My Scheduler Worked. Then I Tried to Starve It.

A clean test that passes can hide the fact that you tested the wrong thing. This is the one where a hostile test caught what a friendly one missed, and it's the most interesting engineering of the day.

**The starvation problem.** Strict priority has no clock. If humans keep arriving, a low-priority agent never wins — it waits forever, because there's always a higher-priority request ahead of it. The standard fix is aging: a request's effective priority improves the longer it waits, so eventually it outranks everything else regardless of its starting priority.

```
effective_priority = base_priority − (seconds_waited × aging_rate)
```

**The architectural catch.** Aging requires reordering requests while they wait — a request's rank in the queue has to change without it ever being dequeued and re-enqueued. That single requirement forced the router from a stateless pass-through into a real scheduler that holds a queue in memory. And it's exactly why the right tool here is a priority queue — `heapq` for now, a Redis sorted set at scale — and not Kafka, which is the reflexive answer whenever an engineer hears "queue." Kafka's entire value proposition is preserving arrival order. Aging's entire value proposition is violating arrival order on purpose, for the requests that have waited long enough to deserve it. Kafka's core guarantee is the exact anti-feature I needed. Not a small mismatch — picking a tool whose main feature is the one thing you don't want.

**The batching constraint.** A naive queue that forwards requests one at a time to vLLM would destroy the 42x batching win from Day 1 — vLLM's continuous batcher needs many in-flight requests to amortize the weight read, and a strict queue serializes them. So the router keeps a fixed number of requests in flight via a semaphore and only queues the overflow. Protect the throughput bus that already works; add a fairness lane on top of it.

**The friendly test passed.** Agent arrives first, gets aged, finishes somewhere in the middle of a mixed batch of humans. Looked like a clean win.

**The hostile test exposed the truth.** I made the test adversarial on purpose: humans arrive first and saturate the semaphore, the agent arrives late at the very bottom of the queue, then more humans stream in *after* the agent. If aging actually works, the agent should claw its way up over time despite starting last.

It didn't look like it worked. The agent finished dead last — 11.3 seconds — behind humans that had arrived 2.4 seconds *after* it did. Aging appeared broken.

**The diagnosis — instrument, don't guess.** Instead of cranking the aging rate up and hoping, I logged every selection event: each waiter's effective priority at that instant, and who the scheduler actually picked. The log proved the selection logic itself was correct — it did eventually pick the agent over waiting humans. The real finding was subtler and more interesting than a bug: uniform aging can't close a gap. If every request ages at the same rate, the *distance* between any two requests' effective priorities stays constant over time — aging shifts the whole distribution down together, it doesn't compress it. The agent only won by outlasting the humans that were already ahead of it when it arrived — 8.3 seconds of pure waiting — not by catching up to anyone. Correct behavior, but slow, and "slow rescue" is functionally indistinguishable from starvation if you're the one waiting.

**The fix — differential rates.** Make agents age faster than humans instead of matching everyone's rate:

```
agent aging_rate = 12 / sec
human aging_rate = 2 / sec
gap closes at (12 − 2) = 10 / sec
```

Now the gap between an agent and the humans ahead of it shrinks at 10 points per second instead of staying frozen. Re-ran the same hostile test — humans-first, agent buried last, more humans after it:

| Request | Finish time | Note |
|---|---|---|
| human-A0 | 2.85s | arrived first — still wins |
| human-A1 | 2.90s | |
| **AGENT** | **5.30s** | rescued from last place to 3rd |
| human-A2 | 5.93s | |
| human-B0–B3 | 7.7–9.5s | latecomers, land behind |

Rescued from dead-last to 3rd place, ahead of five of the six humans in the test, while the two humans that had genuinely arrived first still won outright. Fast rescue and humans-first, both holding at once — the property I actually wanted, not the weaker one the friendly test made me think I already had.

The thesis of the whole exercise: the clean test said "aging works." The hostile test said "aging works, but the rescue is too weak to matter, because uniform rates preserve gaps instead of closing them." That second, truer claim only surfaced because I tried to break the first one. Predict, probe, diagnose the miss, fix, re-verify — the identical loop that caught a truncation bug faking a 71% eval score on Day 2. Same discipline, different layer of the stack.

---

## Demo Values Are Not Production Values, and Saying So Is the Point

Worth being explicit about the numbers above, because burying this is how a demo quietly gets mistaken for a claim. The semaphore is capped at 2 in-flight requests, and the aging rates (12/sec, 2/sec) are test fixtures chosen to force visible contention inside a test that runs in seconds, not settings I'd put in front of real traffic. Real concurrency limits come from the memory-bandwidth math Day 1 and Day 2 already worked out; real aging rates come from tuning against actual arrival patterns, not from what makes a unit test legible.

The gap between "I proved this mechanism works" and "this is tuned for production" is real, and I'd rather name it than let a reader assume the fixture numbers are the recommendation.

## What Ports to Scale

None of today was built as a single-node toy that gets thrown away later. The waiter set plus effective-priority selection is a `heapq` today; it's a Redis sorted set with score updates at scale — same operation, same eventual-consistency story, different storage. The semaphore is per-process concurrency control today; it's per-replica concurrency control behind a load balancer once there's more than one GPU. Single-node is where the logic gets proven, not where it stays.

---

## What's Next

Everything today ran with the router and scheduler doing correct things on a single GPU in a test harness. What I haven't done yet: run the router-plus-scheduler stack under the same concurrency sweep Day 1 ran on raw vLLM, to see what the policy layer costs in aggregate throughput once it's not artificially starved by a semaphore of 2. I expect it to be close to free, the same way priority alone was — but "I expect" is exactly the sentence that's supposed to turn into a measured number before I trust it.
