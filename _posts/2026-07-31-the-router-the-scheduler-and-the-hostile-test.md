---
layout: post
title: "Day 3: A Router and Priority Scheduler for vLLM"
date: 2026-07-31
description: "Serving tokens fast and serving them sensibly are different problems. A streaming translator, a scheduler that isn't Kafka, and a test I had to make adversarial before it showed me what was really going on."
---

[Day 1]({% link _posts/2026-07-15-standing-up-the-inference-seam.md %}) found the number that's been steering everything since: the same GPU, at the same instant, gives a human 14x worse latency and an agent fleet 42x more throughput, depending entirely on who's asking. [Day 2]({% link _posts/2026-07-17-the-quantization-frontier-what-it-actually-costs.md %}) spent a full day inside FP8 and ended with three options: push into FP4, benchmark SGLang, or stop and ship the table.

I didn't do any of them. Day 1's finding was still unresolved: a GPU with no single correct configuration for two kinds of user, and nothing yet that acts on that. Polishing the quantization table further felt pointless while nothing downstream could use it. So today I worked on what Day 1 called for, something that decides per request which model answers and when it runs: a router and a scheduler. Before writing any code I want to explain why both need to exist, because "add a router" is easy to say and easy to build for the wrong reasons.

---

## Why a Router: Requests Need Different Models and Different Effort

Most people building on LLMs make one API call to one model and move on. That's fine with one model, one kind of user and no cost pressure. When humans and agents both use the same GPU pool, all three of those change at once.

That's what I'm building: a platform where humans and agents send requests to the same pool of models, and the infrastructure has to serve them sensibly. It's the control plane underneath a fleet of chatbots rather than a chatbot itself. Model serving is the minimum requirement. The interesting part is everything between a request arriving and a model answering it.

Once you have more than one model, say a cheap fast one and an expensive smart one, every request raises the question of which should answer it. Sending "say hi in three words" to a frontier model wastes money, and sending "prove this theorem" to a tiny model gets you a bad answer. The router makes that decision per request. It sits in front of the models, speaks the same API they do so callers can't tell it's there, and makes three kinds of decision:

**How hard should the model think?** Day 2 put a price on this: thinking mode costs 9.1x the tokens (2421 vs 265) for +5.0 points of accuracy on GSM8K. That's worth it on a hard problem and a waste on "hi," which Day 1 caught spending 108 tokens on a greeting. The router turns thinking mode on or off per request instead of using one default for everything.

**Which model answers?** I ran the same short number-theory proof through both backends today. The small local model struggled to a mostly-correct answer in 3036 tokens, and the frontier model did it cleanly in 498, so the small model used six times the tokens for a worse result. Most requests don't need the frontier model, but a fair number do, and the router has to tell which is which before either backend spends a token.

**It also hides all of this.** The two backends don't speak the same wire format (vLLM is OpenAI-shaped, Anthropic isn't), so the router translates in both directions. From the outside there's one endpoint, and the caller doesn't know which model answered.

That's the difference between having a pile of model endpoints and having a system that sends work to the right one.

## Why a Scheduler: One GPU, Two Users Who Want Opposite Things

Day 1 measured the reason a scheduler is needed before I'd named it. Generating text has two phases that behave very differently. Time-to-first-token blows up under load: Day 1 measured it 14x worse, 16.7ms to 230ms average, going from concurrency 1 to 64. Inter-token latency barely moves, up only 34% over the same range, because the weight read dominates and extra sequences come along mostly for free.

Now think about who's waiting. A person staring at a screen notices every millisecond before the first token. An agent working through a batch job doesn't care if it starts 200ms later, it cares about total throughput. So they want the GPU tuned in opposite directions, and you can't optimize both for the same request at the same time. Day 1's numbers already showed that.

So when a human request and an agent request are both waiting, serving the human first costs the agent almost nothing and saves the human the wait they actually notice. That only happens if something decides whose request goes first, and that's the scheduler. It knows whether a request belongs to a human or an agent (the model-serving layer has no idea, to it every request is just tokens) and orders them accordingly.

## The Shape of It

Put together: a request arrives, the router decides which model and how much thinking, the scheduler decides when it runs relative to everything else, the model serves it, and the answer streams back in the same format whichever backend produced it. The router decides who answers and the scheduler decides when. Model serving sits under both and does the generation. It's very efficient, but it has no idea who's asking or how much the request matters, and that's what the router and scheduler add.

I built three things today, listed here in order of how much each one surprised me.

---

## Making the Router Stream, and Why the Frontier Path Is Harder

Buffered responses are convenient, but real chat streams token by token. Adding streaming to a router in front of two different backends is really two features, because the backends stream differently.

**Local (vLLM) is a straight pipe.** vLLM already emits OpenAI-format SSE chunks, so the router just relays them.

**Frontier (Claude) needs live translation.** Anthropic emits typed events (`message_start`, `content_block_delta`, `message_stop`) that are structured differently from OpenAI's `chat.completion.chunk`:

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

The router has to parse each Anthropic event as it arrives, pull the text out of `content_block_delta`, re-wrap it in OpenAI's chunk shape, and add the `data: [DONE]` sentinel that Claude never sends. Anthropic just closes the connection at `message_stop` and expects you to know that means it's finished.

Before writing the translator, I sent a raw `curl` to the Anthropic endpoint and read the actual event stream instead of trusting my memory of the API. Model APIs change, and checking takes one command, which is a lot cheaper than debugging a translator built on a format that's a few months out of date.

The result: the caller sends `"model": "Qwen3-8B"`, gets back a streamed proof from Claude in vLLM's wire format, and can't tell the difference.

One rough edge I left on purpose: chunk size differs by backend. vLLM emits subword fragments ("Ele" then "ven") because that's how its tokenizer splits text, while Claude sends whole clauses. The router passes through whatever each backend sends instead of normalizing it, since smoothing it out would add latency for a purely cosmetic gain.

---

## vLLM Already Has a Scheduler. So What Am I Building?

This changed how I thought about the rest of the day. As soon as I decided to build a scheduler in front of vLLM, I realized vLLM already has one. Continuous batching, which made Day 1's 42x throughput number possible, is a scheduler. It decides which sequence gets a token on each step, packs the KV cache, and preempts when memory runs short. It's very good and I'm not rebuilding it.

What vLLM's scheduler does is optimize efficiency *within one model*. It can't know whether a request belongs to a human or an agent, it can't see across backends (it has no idea Claude exists), and it has no fairness policy beyond keeping the GPU full.

So the split is simple: **vLLM enforces, the router decides.** vLLM has a `priority` field it will order requests by, but it can't work out what priority a request should get. Mapping users to priorities is policy, and it has to live above vLLM. Most real serving stacks have these two layers, an efficiency engine underneath and a policy layer on top deciding what to prioritize. This is the normal outer layer, nothing exotic.

I didn't want to assume vLLM actually honors the priority field just because the flag exists, so I tested it. I restarted the server with `--scheduling-policy priority`, sent one request with `priority: 0`, and it was accepted without error. But accepting the field doesn't prove anything about ordering, since a single request has nothing to be ordered against. That needs contention.

So I created some. I sent six heavy agent requests to saturate the GPU, then sent one human request dead last, after all six agents were already queued:

| Request | Priority | Launch order | Finish time |
|---|---|---|---|
| human | 0 | last | 0.14s |
| agent (x6) | 10 | first | 11–16s |

The request that arrived last finished roughly 100x faster than the ones ahead of it, because priority let it skip the queue. Day 1's claim that humans and agents want opposite things from the GPU is now backed by a measurement. vLLM's own published benchmarks say priority scheduling costs close to zero aggregate throughput, and that matched what I saw on my hardware.

---

## My Scheduler Worked, Until I Tried to Starve It

A test that passes cleanly can hide the fact that you tested the wrong thing. Here a hostile test caught something a friendly one missed, and it was the most interesting engineering of the day.

**The starvation problem.** Strict priority ignores time. If humans keep arriving, a low-priority agent can wait forever, because there's always a higher-priority request ahead of it. The standard fix is aging: a request's effective priority improves the longer it waits, so eventually it outranks everything else regardless of its starting priority.

```
effective_priority = base_priority − (seconds_waited × aging_rate)
```

**The architectural catch.** Aging means reordering requests while they wait, so a request's position in the queue changes without it being dequeued and re-enqueued. That one requirement turned the router from a stateless pass-through into a real scheduler holding a queue in memory. It's also why the right tool is a priority queue (`heapq` for now, a Redis sorted set at scale) and not Kafka, which is what engineers tend to reach for when they hear "queue." Kafka's main guarantee is preserving arrival order, and aging deliberately breaks arrival order for requests that have waited long enough. Kafka's main feature is exactly what I didn't want here.

**The batching constraint.** A naive queue that sends requests to vLLM one at a time would wipe out Day 1's 42x batching gain, because vLLM's continuous batcher needs many requests in flight to spread the cost of the weight read. So the router keeps a fixed number of requests in flight with a semaphore and only queues the overflow. That keeps the batching that already works and adds fairness on top.

**The friendly test passed.** The agent arrived first, got aged, and finished somewhere in the middle of a mixed batch of humans. It looked fine.

**The hostile test told a different story.** I made it adversarial on purpose: humans arrive first and saturate the semaphore, the agent arrives late at the very bottom of the queue, then more humans stream in *after* the agent. If aging actually works, the agent should claw its way up over time despite starting last.

It didn't look like it. The agent finished last, at 11.3 seconds, behind humans that had arrived 2.4 seconds *after* it. Aging seemed broken.

**The diagnosis.** Instead of turning up the aging rate and hoping, I logged every selection event: each waiting request's effective priority at that moment, and which one the scheduler picked. The log showed the selection logic was correct, and it did eventually pick the agent over waiting humans. The actual problem was more subtle than a bug: uniform aging can't close a gap. If every request ages at the same rate, the *distance* between any two requests' effective priorities stays the same. Everything shifts down together. The agent only won by outlasting the humans already ahead of it when it arrived, which took 8.3 seconds of waiting, not by catching up. That's technically correct, but if you're the one waiting, a rescue that slow feels the same as starvation.

**The fix: different rates.** Make agents age faster than humans instead of matching everyone's rate:

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

The agent went from last place to 3rd, ahead of five of the seven humans, while the two humans who really did arrive first still finished first. So the agent gets rescued quickly and humans still get priority, which is what I actually wanted.

The friendly test said aging works. The hostile test said aging works but the rescue is too weak to matter, because uniform rates keep gaps the same instead of closing them. I only found that out because I tried to break the first result. It's the same process that caught the truncation bug faking a 71% eval score on Day 2: predict, test, figure out the miss, fix, and check again.

---

## Demo Values Are Not Production Values

I want to be clear about the numbers above so nobody mistakes a demo for a recommendation. The semaphore is capped at 2 in-flight requests, and the aging rates (12/sec, 2/sec) are test fixtures chosen to force visible contention inside a test that runs in seconds, not settings I'd put in front of real traffic. Real concurrency limits come from the memory-bandwidth math Day 1 and Day 2 already worked out; real aging rates come from tuning against actual arrival patterns, not from what makes a unit test legible.

Proving the mechanism works is not the same as tuning it for production, and I don't want anyone copying the test values.

## What Ports to Scale

None of this is a single-node toy that gets thrown away later. The waiting set and effective-priority selection are a `heapq` today and become a Redis sorted set with score updates at scale: same operation, different storage. The semaphore is per-process concurrency control today and becomes per-replica concurrency control behind a load balancer once there's more than one GPU. Single-node is just where I'm proving the logic.

---

## What's Next

Everything today ran with the router and scheduler doing correct things on a single GPU in a test harness. What I haven't done yet: run the router-plus-scheduler stack under the same concurrency sweep Day 1 ran on raw vLLM, to see what the policy layer costs in aggregate throughput once it's not artificially starved by a semaphore of 2. I expect it to be close to free, like priority alone was, but I'd like to measure it before I believe that.
