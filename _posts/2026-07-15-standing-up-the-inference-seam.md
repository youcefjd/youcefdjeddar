---
layout: post
title: "Day 1: Baselining vLLM on a Single GPU"
date: 2026-07-15
description: "New series. I put a GPU behind an OpenAI-compatible endpoint, measured before tuning, and found one number that turned a serving project into a distributed-systems project."
---

I've got a GPU under my desk and an idea I can't let go of: host my own models, route between them intelligently, serve both my own chat sessions and a fleet of AI agents off the same hardware. I want it to be a real platform with real constraints: latency budgets, throughput budgets, a quantization strategy, and a router that has to decide when Qwen-thinking is worth the tokens and when a request should just go to OpenAI instead.

This is the first post in that series. Today I only wanted to answer two questions: does this hardware serve tokens, and can I measure it honestly? Whether it's fast, or which serving stack wins, doesn't mean much until I have a baseline.

So today was deliberately boring: no routing logic, no scheduler, no quantization tricks. I stood up an endpoint and stared at a spreadsheet, because otherwise you end up with strong opinions about a GPU you've never measured.

---

## The Two Things Everything Else Depends On

**A seam**: an OpenAI-compatible endpoint that returns tokens. The router, scheduler and delegation logic I build later all hang off `/v1/chat/completions`, so nothing else can start until it exists.

**A measuring instrument**, installed and run *before* touching any tuning knob. If you tune first you end up with opinions, and what I wanted from day one was numbers.

I also picked Qwen3-8B on purpose instead of the bigger model I want in production. An 8B model at bf16 fits this card with room to spare, so if something breaks there's only one suspect. Starting at 27B would give me four suspects at once (serving engine, GPU architecture, checkpoint, VRAM headroom) and turn day one into a debugging session instead of a baseline.

---

## What I Actually Ran

1. **Toolchain check.** PyTorch reported CUDA capability `(12, 0)`, so Blackwell kernels were compiled in and the driver and runtime were fine. Stock wheels, no source build.
2. **Serving.** `vllm serve Qwen/Qwen3-8B --port 8000`. FlashAttention 2 picked automatically, FlashInfer sampler JIT-compiled clean, CUDA graphs on, memory utilization at the ~0.9 default.
3. **Seam confirmation.** `curl` against `/v1/chat/completions` returned tokens.
4. **Instrumentation.** Installed `genai-perf` (NVIDIA's Triton benchmarking tool) to standardize TTFT, inter-token latency, and throughput under synthetic load, instead of eyeballing timestamps.
5. **The sweep.** Concurrency 1 / 4 / 16 / 64. Synthetic 128 tokens in, 128 out. Thinking mode on (Qwen3's default). bf16 throughout. Concurrency was the only variable.

| Concurrency | TTFT avg (ms) | TTFT p99 (ms) | ITL avg (ms) | Per-user tok/s | Aggregate tok/s |
|---|---|---|---|---|---|
| 1 | 16.68 | 18.25 | 10.46 | 95.61 | 95.15 |
| 4 | 24.97 | 27.93 | 11.05 | 90.49 | 298.52 |
| 16 | 119.04 | 171.36 | 11.48 | 87.11 | 1297.44 |
| 64 | 229.85 | 377.20 | 14.04 | 71.42 | 4046.58 |

Two of these four rows tell opposite stories about the same GPU at the same moment, and that contradiction turned out to be the main finding of the day.

---

## The Physics: Decode Is Limited by Memory

```
Qwen3-8B @ bf16    = 16.4 GB of weights
RTX 5090 bandwidth = 1792 GB/s
theoretical floor  = 16.4 / 1792 = 9.15 ms/token
measured ITL (c1)  = 10.46 ms/token
→ 87% of the theoretical floor
```

Generating one decode token means reading *every weight in the model* from memory into the compute units, all 16.4 GB, for every token. The math itself is trivial by comparison, and the tensor cores spend most of their time waiting for data. So token rate depends on how fast the GPU can stream weights out of memory, not on compute speed. 9.15 ms/token is the physical floor for this model on this card, and I measured 10.46, which is 87% of the way there.

`nvidia-smi` would have shown a busy GPU the whole run, but it was mostly busy waiting. "GPU utilization: 95%" is misleading here. Memory bandwidth utilization is the number to watch.

That changes the whole optimization strategy. **Decode can't be tuned much faster on this hardware.** No flag, scheduler tweak or batch size will recover the remaining 13% in any meaningful way. That leaves two structural levers:

- **Move less data per token** with quantization. FP8 roughly halves the weight read to ~8 GB. This is when it clicked for me that quantization is a *serving* technique as well as a way to fit a bigger model in less VRAM.
- **Read fewer weights per token** with mixture-of-experts (activate 3B of 35B params instead of all of them) or speculative decoding (verify several tokens for the cost of roughly one weight read).

The second point is also why the bigger MoE model I want in production should *outrun* a same-sized dense model on this card. I haven't benchmarked that yet. It's a prediction from the bandwidth math, which is at least better than a hunch.

---

## Prefill and Decode Behave Like Two Different Machines

TTFT at concurrency 1 was 16.68 ms to prefill 128 tokens. ITL was 10.46 ms to decode a single token. Per token, prefill was roughly 130x cheaper.

Prefill processes the whole prompt as one matrix operation: all 128 positions at once, in one pass over the weights, with the tensor cores fully busy. That's compute-bound, which is what this hardware was built for. Decode is a sequential chain, since token *n+1* can't start until token *n* exists, so each pass over 16.4 GB of weights produces exactly one token. That's bandwidth-bound, and the compute is nearly free.

So the same model in the same run has two phases with opposite resource profiles. I'm writing this down now because I'll need it as a control later. NVFP4 should speed up prefill a lot and leave decode roughly flat, since FP4 speeds up compute and only prefill is compute-bound. Without today's clean bf16 numbers I'd have nothing to check that against.

---

## Batching Is Almost Free, Because the Expensive Part Is Already Paid For

Aggregate throughput went from 95 to 4047 tok/s across the sweep, a 42.5x gain for a 64x increase in concurrency, while per-user throughput dropped only 25% (95.6 to 71.4 tok/s).

That's close to a free lunch, and there's a mechanical reason for it. Every forward pass reads the full 16.4 GB of weights regardless of batch size. At concurrency 1 I pay 16.4 GB for one token, and at concurrency 16 I pay the *same* 16.4 GB for sixteen. Batching doesn't make the GPU faster, it spreads a fixed cost over more requests.

ITL shows this clearly: 10.46 → 14.04 ms across a 64x jump in concurrency, only +34%. If batching were expensive, ITL would have climbed along with concurrency. It barely moved because the weight read dominates and the extra sequences come along almost for free. That's what continuous batching gives you over static batching: new requests join a running batch mid-flight instead of waiting for the slowest sequence to finish, so the GPU doesn't sit idle between batches.

The step from 4 to 16 concurrency was actually *superlinear*, a 4.35x throughput gain from a 4x concurrency increase (109% efficiency). That's not a measurement error. When you're bandwidth-bound with idle compute, each extra sequence costs almost nothing, and fixed per-step overhead gets spread thinner as the batch grows. From 16 to 64, efficiency dropped to 78% as headroom ran out and prefill contention started to show. That's useful input for whatever scheduler I build next.

---

## The Number That Matters Most Today

TTFT went from 16.7 ms to 229.85 ms across the sweep, 13.8x worse. Batching doesn't help this metric, it actively hurts it, and the reason is the mirror image of the batching story. Prefill is compute-bound, so concurrent prefills really do compete for a limited resource, and there's no fixed cost to spread around when the tensor cores themselves are the bottleneck. They just queue behind each other.

The spread at concurrency 64 shows it well: TTFT ranged from 22.34 ms to 377.20 ms *within the same run*. The minimum is basically the uncontended concurrency-1 number, the first request in while the GPU was idle. The maximum spent most of its time in a queue. A 17x spread inside one run means the queue has no ordering policy at all.

This is the main finding of the day, and it's why the project became a distributed-systems problem and not just a serving one:

| Seat | Metric | c1 → c64 |
|---|---|---|
| A human waiting on a chat reply | TTFT | 16.7 → 230 ms avg / 377 ms p99 — **14x worse** |
| An agent fleet doing batch work | Aggregate throughput | 95 → 4047 tok/s — **42x better** |

Same GPU, same moment, and whether it's doing well depends entirely on who you ask. Neither one is wrong, and no single configuration of this GPU makes both happy. The setting that helps the agent fleet is the same one that ruins my chat latency, and that follows directly from the compute-bound vs bandwidth-bound split above.

So admission control, priority classes, per-principal quotas and backpressure aren't extras I'm adding because I like them. When one GPU serves two kinds of users with conflicting needs, you need them. I assumed I'd want some of that eventually. I didn't expect the numbers to require it on day one.

---

## The Ten-Token Question That Cost 119 Tokens

Small thing, but it'll matter once I start comparing quantization levels. I sent the model "hi" and got back `completion_tokens: 119`. The actual answer, "Hello! 😊 How can I assist you today?", is about eleven tokens. Qwen3 spent roughly 108 tokens *thinking about how to say hello.*

That matters in two ways. First, it's a benchmarking trap. Reasoning mode inflates tokens per request, time to useful output, and cost by something like 10x on trivial prompts, and doesn't affect TTFT at all, because prefill doesn't care what happens after the first token. If I don't hold thinking mode constant across every future benchmark row, I risk measuring a thinking-mode difference and calling it a quantization difference. Today's rows: thinking on, written down so future rows can match it or deliberately diverge.

Second, it's an input for routing. The router I build won't only choose local vs hosted or small vs large. It also has to choose *how hard to think*, and the "hi" example shows that thinking isn't free just because it happens automatically.

It's also just funny. 108 tokens to work out how to say hello.

---

## The One Setup Detail Worth Publishing on Its Own

A toolchain note, because I lost real time on this and most advice online is out of date. The old guidance is to avoid `nvidia-cuda-toolkit` because it ships CUDA 12.4, which predates Blackwell, and that's still true. But the follow-up advice to build PyTorch from source with `TORCH_CUDA_ARCH_LIST=12.0` is no longer needed. What worked for me: pulling `cuda-toolkit` 13.1.1 from NVIDIA's own repo, one source, never mixed with the distro package; `nvcc` at `/usr/local/cuda/bin` needing an explicit `PATH` export; `build-essential` installed because Triton JIT-compiles a host-side C shim at runtime. Stock wheels, no source build. It probably deserves its own post, but I'm noting it here because it's why today wasn't lost to toolchain problems.

---

## What I'm Predicting for Next Time

There was nothing to pre-register today, since this *is* the baseline. But the point of a baseline is to make the next claim testable, so here it is before I run it:

**FP8 should roughly halve ITL and barely move TTFT.**

Decode is bandwidth-bound at 87% of the theoretical floor, so halving the bytes moved per token (16.4 GB → ~8 GB) should nearly halve time per token. Prefill is compute-bound, and FP8 doesn't reduce the amount of computation, only the numeric precision, so TTFT should be roughly unchanged, modulo whatever throughput difference the tensor cores have at lower precision.

If ITL halves and TTFT holds, my mental model works on my own hardware and not just in blog posts. If it doesn't, that's actually more useful, because it means something in my model is wrong and I get to find out what.

---

Next up: running that FP8 prediction for real, and sketching what a scheduler that respects the human-vs-agent split above would need to look like.

I started today just wanting to know whether the GPU worked. It does, and it also handed me a distributed-systems problem I wasn't looking for, which isn't bad for one afternoon.
