---
layout: post
title: "Standing Up the Seam: Day 1 of Serving Qwen to a Fleet of Humans and Agents"
date: 2026-07-15
description: "New series. I put a GPU behind an OpenAI-compatible endpoint, measured before tuning, and found one number that turned a serving project into a distributed-systems project."
---

I've got a GPU sitting under my desk and an idea I can't quite let go of: host my own models, route between them intelligently, and serve both my own chat sessions and a fleet of AI agents off the same hardware.

Not a toy. A real platform, with the constraints a real platform has — latency budgets, throughput budgets, a quantization strategy, a router that has to decide when Qwen-thinking is worth the tokens and when the request should just go to OpenAI instead.

This is the first post in that series. It'll take weeks, maybe longer. Today was day one, and day one had exactly one job: does this hardware serve tokens, and can I measure it honestly?

Not "is it fast." Not "which serving stack wins." Those questions are meaningless without a baseline, and I don't have one yet.

---

## The Two Things Everything Else Depends On

Before any tuning, two things had to exist:

**A seam** — an OpenAI-compatible endpoint that returns tokens. Everything I build on top of this — router, scheduler, delegation logic — hangs off `/v1/chat/completions`. Until that endpoint exists, there's no data plane to abstract over.

**A measuring instrument** — installed and run *before* I touched a single tuning knob. This ordering isn't fussiness. Tune first and you acquire opinions. Measure first and you acquire numbers. Numbers are the actual deliverable of day one.

One more deliberate choice: I started with Qwen3-8B, not the bigger Qwen I actually want to run in production. An 8B model at bf16 fits on this card with room to spare, which means if something breaks, there's exactly one suspect. Starting at 27B or larger would've handed me four suspects at once — the serving engine, the GPU architecture, the checkpoint, VRAM headroom — and turned day one into a debugging session instead of a baseline. Small model first isn't timidity. It's failure isolation.

---

## What I Actually Ran

1. **Toolchain check.** PyTorch reported CUDA capability `(12, 0)` — Blackwell kernels compiled in, driver and runtime chain intact. Passed on stock wheels, no source build required.
2. **Serving.** `vllm serve Qwen/Qwen3-8B --port 8000`. vLLM picked FlashAttention 2 out of its candidate backends, the FlashInfer sampler JIT-compiled clean, CUDA graphs enabled, memory utilization left at the ~0.9 default.
3. **Seam confirmation.** `curl` against `/v1/chat/completions`. Tokens came back. Seam exists.
4. **Instrumentation.** Installed `genai-perf` (NVIDIA's Triton benchmarking tool) to standardize time-to-first-token, inter-token latency, and throughput under synthetic load, instead of eyeballing timestamps myself.
5. **The sweep.** Concurrency 1 / 4 / 16 / 64. Synthetic 128 tokens in, 128 out. Thinking mode on (Qwen3's default). bf16 throughout. One variable moved: concurrency.

| Concurrency | TTFT avg (ms) | TTFT p99 (ms) | ITL avg (ms) | Per-user tok/s | Aggregate tok/s |
|---|---|---|---|---|---|
| 1 | 16.68 | 18.25 | 10.46 | 95.61 | 95.15 |
| 4 | 24.97 | 27.93 | 11.05 | 90.49 | 298.52 |
| 16 | 119.04 | 171.36 | 11.48 | 87.11 | 1297.44 |
| 64 | 229.85 | 377.20 | 14.04 | 71.42 | 4046.58 |

Four rows. Two of them tell opposite stories about the same GPU at the same instant. That contradiction turned out to be the actual finding of the day, and I'll get to it.

---

## The Physics: Decode Is a Memory Problem, Not a Compute Problem

Here's the arithmetic that made everything else make sense:

```
Qwen3-8B @ bf16    = 16.4 GB of weights
RTX 5090 bandwidth = 1792 GB/s
theoretical floor  = 16.4 / 1792 = 9.15 ms/token
measured ITL (c1)  = 10.46 ms/token
→ 87% of the theoretical floor
```

Generating one decode token means reading *every weight in the model* from memory into the compute units. All 16.4 GB of it, every single token. The actual math involved is trivial by comparison — the tensor cores are mostly sitting there waiting for data to show up. So the token rate isn't a function of how fast the GPU computes. It's a function of how fast it can stream weights off memory. 9.15 ms/token is the physical floor for this model on this card. I measured 10.46. That's 87% of the ceiling that actually binds.

I want to flag why that framing matters, because `nvidia-smi` would have shown a busy GPU during this whole run. Busy doing what, though? Waiting. "GPU utilization: 95%" is a number that lies to you in this regime. Memory bandwidth utilization is the number that tells the truth.

And that number forks the whole optimization strategy. **Decode cannot be tuned faster on this hardware.** There's no flag, no scheduler tweak, no batch size that recovers the remaining 13% into anything meaningful. Only two levers exist, and both are structural:

- **Move less data per token** — quantization. FP8 roughly halves the weight read to ~8 GB. This is the thing that made it click for me that quantization is a *serving* technique, not just a trick to fit a bigger model in less VRAM.
- **Read fewer weights per token** — mixture-of-experts (activate 3B of 35B params instead of all of them) or speculative decoding (verify several tokens for the cost of roughly one weight read).

That second bullet is also, not coincidentally, why the bigger MoE model I actually want to run in production should *outrun* a same-sized dense model on this exact card. Not a benchmark claim yet — a prediction from the bandwidth equation, which I like a lot better than a vibe.

---

## Prefill and Decode Are Two Different Machines Wearing One Costume

TTFT at concurrency 1 was 16.68 ms to prefill 128 tokens. ITL was 10.46 ms to decode a single token. Per token, prefill was roughly 130x cheaper.

The reason: prefill processes the whole prompt as one matrix operation — all 128 positions at once, one pass over the weights, tensor cores actually saturated for once. It's compute-bound, which is the regime this hardware was built for. Decode is a sequential chain — token *n+1* literally cannot start until token *n* exists — so one pass over 16.4 GB of weights buys you exactly one token. It's bandwidth-bound, and the compute is nearly free.

Same table, same model, same run — two regimes with inverted resource profiles. I'm writing this down now because it's the control I'll need later: NVFP4 should speed up prefill substantially and leave decode roughly flat, since FP4 attacks compute and only prefill is compute-bound. Without today's clean bf16 split of the two regimes, that future result would just be an anecdote. With it, it's a confirmed prediction.

---

## Batching Is (Almost) Free, Because the Expensive Part Is Already Paid For

Aggregate throughput went from 95 to 4047 tok/s across the sweep — a 42.5x gain for a 64x increase in concurrency. Per-user throughput only dropped 25%, from 95.6 to 71.4 tok/s.

That ratio looks like a free lunch, and it basically is, for a specific mechanical reason: every forward pass reads the full 16.4 GB of weights regardless of batch size. At concurrency 1, I pay 16.4 GB to produce one token. At concurrency 16, I pay the *same* 16.4 GB to produce sixteen tokens. Batching doesn't make the GPU faster — it amortizes a fixed cost across more requests riding the same bus trip.

ITL is the tell: 10.46 → 14.04 ms across a 64x jump in concurrency. Only +34%. If batching were expensive, ITL would've climbed in step with concurrency. It barely moved, because the weight read dominates and the extra sequences are along for a ride that was happening anyway. This is what continuous batching buys you over static batching — new requests join a running batch mid-flight instead of waiting for the slowest sequence in the batch to finish, so the GPU never idles between batches.

The step from 4 to 16 concurrency was actually *superlinear* — a 4.35x throughput gain from a 4x concurrency increase, 109% efficiency. That's not a measurement error. In a bandwidth-bound regime with idle compute, each additional sequence added to a batch costs next to nothing incrementally, and fixed per-step overhead (batch formation, CUDA graph replay, sampler dispatch) spreads thinner as the batch grows. The step from 16 to 64 dropped back to 78% efficiency — headroom running out, prefill contention starting to bite. That inflection point is a real design input for whatever scheduler I build next, not a footnote.

---

## The Number That Matters Most Today

TTFT went from 16.7 ms to 229.85 ms across the sweep — 13.8x worse. That's exactly the metric batching does *not* help, and actively harms, and the mechanism is the mirror image of the batching story above: prefill is compute-bound, and compute-bound work genuinely competes for a finite resource. There's no fixed cost to amortize when the tensor cores themselves are the bottleneck. Concurrent prefills queue behind each other.

The spread at concurrency 64 makes this vivid: TTFT ranged from a 22.34 ms minimum to a 377.20 ms maximum, within the *same run*. The minimum is basically the uncontended concurrency-1 number — first request in, GPU idle. The maximum is a request that sat in a queue. A 17x spread inside one run isn't noise. It's a queue with no discipline.

So here's the actual finding of the day, and it's the reason this whole project turns into a distributed-systems problem and not just a serving problem:

| Seat | Metric | c1 → c64 |
|---|---|---|
| A human waiting on a chat reply | TTFT | 16.7 → 230 ms avg / 377 ms p99 — **14x worse** |
| An agent fleet doing batch work | Aggregate throughput | 95 → 4047 tok/s — **42x better** |

Same GPU. Same instant. Opposite verdicts, depending entirely on who's asking.

Neither one of those principals is wrong. There is no single configuration of this GPU that satisfies both. The knob that makes the agent fleet happy is the exact knob that ruins my own chat latency, and there's no getting around that with cleverness — it's structural, it comes straight out of the compute-bound-vs-bandwidth-bound split above.

Which means admission control, priority classes, per-principal quotas, backpressure — these aren't architectural taste I'm layering on for fun. They're the only available response to a resource that has no single correct configuration when it's serving two different classes of principal off the same GPU at the same time. I went into today assuming I'd need some of that eventually. I didn't expect to get the actual number that *demands* it on day one.

---

## The Ten-Token Question That Cost 119 Tokens

Small thing, but it'll matter a lot once I start comparing quantization levels: I sent the model "hi" and it came back with `completion_tokens: 119`. The actual answer — "Hello! 😊 How can I assist you today?" — is about eleven tokens. Qwen3 spent roughly 108 tokens *thinking about how to say hello.*

Two things fall out of that. First, a benchmarking trap: reasoning mode inflates tokens-per-request, time-to-useful-output, and cost by something like 10x on trivial prompts, and it doesn't touch TTFT at all — prefill doesn't care what happens after the first token. If I don't hold thinking mode constant across every row of every future benchmark, I risk quietly measuring a thinking-mode difference and calling it a quantization difference. Today's rows: thinking on, written down so future rows can match it or deliberately diverge from it.

Second, it's a routing input. The router I eventually build isn't only choosing local-vs-hosted or small-vs-large-model. It's also choosing *how hard to think*, and "hi" is proof that deliberation isn't free just because it's automatic. That's a cost/latency/quality tradeoff sitting right there in the default behavior, and today I got to see exactly what it costs when it isn't warranted.

---

## The One Setup Detail Worth Publishing on Its Own

Quick toolchain note, because I burned real time on this and the internet's advice on it is stale: the old guidance is to avoid `nvidia-cuda-toolkit` because it ships CUDA 12.4, which predates Blackwell. True, but the follow-on advice — build PyTorch from source with `TORCH_CUDA_ARCH_LIST=12.0` — is now dead. What actually worked was pulling `cuda-toolkit` 13.1.1 from NVIDIA's own repo, one source, never mixed with the distro package, `nvcc` at `/usr/local/cuda/bin` needing an explicit `PATH` export, and `build-essential` installed because Triton JIT-compiles a host-side C shim at runtime. Stock wheels. No source build. That's a whole separate post, but I wanted it on the record here since it's the reason today wasn't a lost day fighting the toolchain.

---

## What I'm Predicting for Next Time

I didn't pre-register anything today — there was nothing to predict against yet, this *is* the baseline. But the baseline exists specifically so the next claim is falsifiable, and here it is, on the record before I run it:

**FP8 should roughly halve ITL and barely move TTFT.**

The reasoning: decode is bandwidth-bound at 87% of the theoretical floor, so halving the bytes moved per token (16.4 GB → ~8 GB) should nearly halve the time per token. Prefill is compute-bound, and FP8 doesn't reduce the amount of computation, just the numeric precision — so TTFT should be roughly untouched, modulo whatever throughput difference the tensor cores have at lower precision.

If ITL halves and TTFT holds, my mental model is correct on my own silicon, not just recited from a blog post. If it doesn't, that's actually the more valuable outcome — it means something in the model is wrong, and finding out what is worth more than a confirmed guess.

---

Next up: running that FP8 prediction for real, and starting to sketch what a scheduler that actually respects the human-vs-agent split above would need to look like.
