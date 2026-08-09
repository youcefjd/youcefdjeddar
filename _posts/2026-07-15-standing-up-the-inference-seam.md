---
layout: post
title: "Standing Up the Seam: Day 1 of Serving Qwen to a Fleet of Humans and Agents"
date: 2026-07-15
description: "New series. I put a GPU behind an OpenAI-compatible endpoint, measured before tuning, and found one number that turned a serving project into a distributed-systems project."
---

I've got a GPU under my desk and an idea I can't let go of: host my own models, route between them intelligently, serve both my own chat sessions and a fleet of AI agents off the same hardware.

Not a toy. A real platform, with the constraints a real platform has — latency budgets, throughput budgets, a quantization strategy, a router that has to decide when Qwen-thinking is worth the tokens and when a request should just go to OpenAI instead.

This is post one of that series. Today had exactly one job: does this hardware serve tokens, and can I measure it honestly?

Not "is it fast." Not "which serving stack wins." Those questions are meaningless without a baseline, and I don't have one yet.

So today was deliberately unglamorous. No routing logic, no scheduler, no clever quantization trick. Stand up an endpoint, stare at a spreadsheet. Skip that step and you end up with strong opinions about a GPU you've never actually measured.

---

## The Two Things Everything Else Depends On

**A seam** — an OpenAI-compatible endpoint that returns tokens. Router, scheduler, delegation logic — everything I build later hangs off `/v1/chat/completions`. No endpoint, no data plane to abstract over.

**A measuring instrument** — installed and run *before* touching a single tuning knob. Tune first and you acquire opinions. Measure first and you acquire numbers. Numbers are the deliverable of day one.

One more deliberate choice: Qwen3-8B, not the bigger model I actually want in production. An 8B model at bf16 fits this card with room to spare, so if something breaks, there's exactly one suspect. Starting at 27B hands me four suspects at once — serving engine, GPU architecture, checkpoint, VRAM headroom — and turns day one into a debugging session instead of a baseline. Small model first isn't timidity. It's failure isolation.

---

## What I Actually Ran

1. **Toolchain check.** PyTorch reported CUDA capability `(12, 0)` — Blackwell kernels compiled in, driver and runtime intact. Stock wheels, no source build.
2. **Serving.** `vllm serve Qwen/Qwen3-8B --port 8000`. FlashAttention 2 picked automatically, FlashInfer sampler JIT-compiled clean, CUDA graphs on, memory utilization at the ~0.9 default.
3. **Seam confirmation.** `curl` against `/v1/chat/completions`. Tokens came back. Seam exists.
4. **Instrumentation.** Installed `genai-perf` (NVIDIA's Triton benchmarking tool) to standardize TTFT, inter-token latency, and throughput under synthetic load, instead of eyeballing timestamps.
5. **The sweep.** Concurrency 1 / 4 / 16 / 64. Synthetic 128 tokens in, 128 out. Thinking mode on (Qwen3's default). bf16 throughout. One variable moved: concurrency.

| Concurrency | TTFT avg (ms) | TTFT p99 (ms) | ITL avg (ms) | Per-user tok/s | Aggregate tok/s |
|---|---|---|---|---|---|
| 1 | 16.68 | 18.25 | 10.46 | 95.61 | 95.15 |
| 4 | 24.97 | 27.93 | 11.05 | 90.49 | 298.52 |
| 16 | 119.04 | 171.36 | 11.48 | 87.11 | 1297.44 |
| 64 | 229.85 | 377.20 | 14.04 | 71.42 | 4046.58 |

Four rows. Two of them tell opposite stories about the same GPU at the same instant. That contradiction is the actual finding of the day.

---

## The Physics: Decode Is a Memory Problem, Not a Compute Problem

```
Qwen3-8B @ bf16    = 16.4 GB of weights
RTX 5090 bandwidth = 1792 GB/s
theoretical floor  = 16.4 / 1792 = 9.15 ms/token
measured ITL (c1)  = 10.46 ms/token
→ 87% of the theoretical floor
```

Generating one decode token means reading *every weight in the model* from memory into the compute units — all 16.4 GB, every single token. The actual math is trivial by comparison; the tensor cores mostly sit there waiting for data to show up. Token rate isn't a function of compute speed. It's a function of how fast the GPU can stream weights off memory. 9.15 ms/token is the physical floor for this model on this card. I measured 10.46. 87% of the ceiling that actually binds.

`nvidia-smi` would've shown a busy GPU this whole run. Busy doing what, though? Waiting. "GPU utilization: 95%" lies to you in this regime. Memory bandwidth utilization is the number that tells the truth.

That number forks the entire optimization strategy. **Decode cannot be tuned faster on this hardware.** No flag, no scheduler tweak, no batch size recovers the remaining 13% into anything meaningful. Two levers, both structural:

- **Move less data per token** — quantization. FP8 roughly halves the weight read to ~8 GB. This is the thing that made it click for me that quantization is a *serving* technique, not just a way to fit a bigger model in less VRAM.
- **Read fewer weights per token** — mixture-of-experts (activate 3B of 35B params instead of all of them) or speculative decoding (verify several tokens for the cost of roughly one weight read).

That second bullet is also, not coincidentally, why the bigger MoE model I actually want in production should *outrun* a same-sized dense model on this exact card. Not a benchmark claim yet — a prediction from the bandwidth equation, which beats a vibe.

---

## Prefill and Decode Are Two Different Machines Wearing One Costume

TTFT at concurrency 1 was 16.68 ms to prefill 128 tokens. ITL was 10.46 ms to decode a single token. Per token, prefill was roughly 130x cheaper.

Prefill processes the whole prompt as one matrix operation — all 128 positions at once, one pass over the weights, tensor cores actually saturated. Compute-bound, the regime this hardware was built for. Decode is a sequential chain — token *n+1* can't start until token *n* exists — so one pass over 16.4 GB of weights buys exactly one token. Bandwidth-bound, and the compute is nearly free.

Same table, same model, same run — two regimes with inverted resource profiles. Writing it down now because it's the control I'll need later: NVFP4 should speed up prefill substantially and leave decode roughly flat, since FP4 attacks compute and only prefill is compute-bound. Without today's clean bf16 split, that future result is just an anecdote. With it, it's a confirmed prediction.

---

## Batching Is (Almost) Free, Because the Expensive Part Is Already Paid For

Aggregate throughput went from 95 to 4047 tok/s across the sweep — a 42.5x gain for a 64x increase in concurrency. Per-user throughput dropped only 25%, 95.6 to 71.4 tok/s.

That ratio looks like a free lunch. It basically is, for a specific mechanical reason: every forward pass reads the full 16.4 GB of weights regardless of batch size. At concurrency 1, I pay 16.4 GB to produce one token. At concurrency 16, I pay the *same* 16.4 GB to produce sixteen. Batching doesn't make the GPU faster — it amortizes a fixed cost across more requests riding the same bus trip.

ITL is the tell: 10.46 → 14.04 ms across a 64x jump in concurrency. Only +34%. If batching were expensive, ITL would've climbed in step with concurrency. It barely moved, because the weight read dominates and the extra sequences ride along for free. This is what continuous batching buys over static batching — new requests join a running batch mid-flight instead of waiting for the slowest sequence to finish, so the GPU never idles between batches.

The step from 4 to 16 concurrency was actually *superlinear* — 4.35x throughput gain from a 4x concurrency increase, 109% efficiency. Not a measurement error. In a bandwidth-bound regime with idle compute, each additional sequence costs next to nothing incrementally, and fixed per-step overhead spreads thinner as the batch grows. The step from 16 to 64 dropped to 78% efficiency — headroom running out, prefill contention starting to bite. Real design input for whatever scheduler comes next, not a footnote.

---

## The Number That Matters Most Today

TTFT went from 16.7 ms to 229.85 ms across the sweep — 13.8x worse. That's exactly the metric batching does *not* help, and actively harms. The mechanism is the mirror image of the batching story: prefill is compute-bound, and compute-bound work genuinely competes for a finite resource. No fixed cost to amortize when the tensor cores themselves are the bottleneck. Concurrent prefills queue behind each other.

The spread at concurrency 64 makes this vivid: TTFT ranged from a 22.34 ms minimum to a 377.20 ms maximum, *within the same run*. The minimum is basically the uncontended concurrency-1 number — first request in, GPU idle. The maximum sat in a queue. A 17x spread inside one run isn't noise. It's a queue with no discipline.

Here's the actual finding of the day, and it's why this whole project turns into a distributed-systems problem and not just a serving problem:

| Seat | Metric | c1 → c64 |
|---|---|---|
| A human waiting on a chat reply | TTFT | 16.7 → 230 ms avg / 377 ms p99 — **14x worse** |
| An agent fleet doing batch work | Aggregate throughput | 95 → 4047 tok/s — **42x better** |

Same GPU. Same instant. Opposite verdicts, depending entirely on who's asking.

Verdict: there is no verdict. That's the finding.

Neither principal is wrong. No single configuration of this GPU satisfies both. The knob that makes the agent fleet happy is the exact knob that ruins my own chat latency — structural, straight out of the compute-bound-vs-bandwidth-bound split above.

Which means admission control, priority classes, per-principal quotas, backpressure aren't architectural taste I'm layering on for fun. They're the only available response to a resource with no single correct configuration when it's serving two classes of principal off the same GPU at the same time. I assumed I'd need some of that eventually. I didn't expect the number that *demands* it on day one.

---

## The Ten-Token Question That Cost 119 Tokens

Small thing, but it'll matter once I start comparing quantization levels: I sent the model "hi" and it came back with `completion_tokens: 119`. The actual answer — "Hello! 😊 How can I assist you today?" — is about eleven tokens. Qwen3 spent roughly 108 tokens *thinking about how to say hello.*

Two things fall out of that. First, a benchmarking trap: reasoning mode inflates tokens-per-request, time-to-useful-output, and cost by something like 10x on trivial prompts, and doesn't touch TTFT at all — prefill doesn't care what happens after the first token. If I don't hold thinking mode constant across every future benchmark row, I risk measuring a thinking-mode difference and calling it a quantization difference. Today's rows: thinking on, written down so future rows can match it or deliberately diverge.

Second, it's a routing input. The router I eventually build isn't only choosing local-vs-hosted or small-vs-large. It's choosing *how hard to think*, and "hi" is proof that deliberation isn't free just because it's automatic. A cost/latency/quality tradeoff sitting right there in the default behavior.

108 tokens to decide how to say hello. The model isn't stupid. It's, in this one specific moment, extremely committed to a bit nobody asked for.

---

## The One Setup Detail Worth Publishing on Its Own

Toolchain note, because I burned real time on this and the internet's advice is stale: the old guidance is to avoid `nvidia-cuda-toolkit` because it ships CUDA 12.4, which predates Blackwell. True. But the follow-on advice — build PyTorch from source with `TORCH_CUDA_ARCH_LIST=12.0` — is now dead. What actually worked: pulling `cuda-toolkit` 13.1.1 from NVIDIA's own repo, one source, never mixed with the distro package; `nvcc` at `/usr/local/cuda/bin` needing an explicit `PATH` export; `build-essential` installed because Triton JIT-compiles a host-side C shim at runtime. Stock wheels. No source build. Worth its own post — noting it here because it's the reason today wasn't lost to toolchain hell.

---

## What I'm Predicting for Next Time

No pre-registration today — there was nothing to predict against yet, this *is* the baseline. But the baseline exists so the next claim is falsifiable. Here it is, on the record before I run it:

**FP8 should roughly halve ITL and barely move TTFT.**

Decode is bandwidth-bound at 87% of the theoretical floor, so halving the bytes moved per token (16.4 GB → ~8 GB) should nearly halve time per token. Prefill is compute-bound, and FP8 doesn't reduce the amount of computation, just the numeric precision — so TTFT should be roughly untouched, modulo whatever throughput difference the tensor cores have at lower precision.

If ITL halves and TTFT holds, my mental model is correct on my own silicon, not just recited from a blog post. If it doesn't, that's the more valuable outcome — it means something in the model is wrong, and finding out what is worth more than a confirmed guess.

---

Next up: running that FP8 prediction for real, and sketching what a scheduler that respects the human-vs-agent split above would need to look like.

I came in today just wanting to know if the GPU worked. It worked. It also handed me a distributed-systems problem I wasn't shopping for. Good trade for one afternoon.
