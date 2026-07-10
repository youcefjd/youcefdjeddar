---
layout: post
title: "What I Learned Benchmarking Llama 3 8B Three Ways"
date: 2026-07-10
description: "The results were satisfying. The bug I hit along the way taught me more."
---

I wanted to actually understand how model serving works, not just repeat facts I'd read in a blog post.

So this week: benchmark the same 8B-parameter model, served three different ways, and figure out — not just observe — why the numbers come out the way they do.

The results were satisfying.

The bug I hit along the way taught me more.

---

## Why This Project, and Why Now

Before touching any code, I spent time building a mental model of how LLM inference actually works, mechanically: what a KV cache is, why inference splits into two phases with different performance characteristics, and what quantization is actually doing to the numbers under the hood.

I wanted vocabulary and intuition *before* I started deploying things.

Otherwise "FP8 is faster" is just a fact I memorized. Not something I can reason about when a real system misbehaves.

That distinction mattered a lot by the end of this project. More on that shortly.

---

## The Mental Model, Briefly

Every LLM inference request has two phases, and they have genuinely different performance profiles.

**Prefill** — the model processes your entire input prompt at once, all tokens, in parallel, through every layer. This is **compute-bound**: the GPU's tensor cores are the bottleneck, not memory.

**Decode** — the model generates output one token at a time, sequentially. Each step reloads the same weight matrices from GPU memory to do a comparatively tiny amount of math. This is **memory-bound**: the compute cores mostly sit idle, waiting for data to show up.

Every generated token also produces a Key and Value vector, cached in GPU memory so future tokens can attend back to it without recomputing it from scratch. That KV cache grows by one entry per token, per layer. For a 32-layer model generating a long response, that's a lot of accumulating state — and it's a big part of why long conversations get progressively more expensive to serve.

Quantization — dropping weights from 16-bit floating point down to 8-bit — helps *both* phases, but for two completely unrelated hardware reasons:

- It helps **prefill** because GPU tensor cores are physically faster at FP8 arithmetic than FP16 — simpler circuitry, more operations per clock cycle. A compute-throughput fact.
- It helps **decode** because FP8 weights are half the bytes of FP16 weights, so there's half as much data to move from memory to the compute cores on every token generated. A data-transfer fact.

Two separate mechanisms. One precision change. Both bottlenecks addressed.

That's the idea I wanted to actually *see* in real numbers — not accept as a claim from a blog post.

---

## The Setup

I served Llama 3 8B Instruct three ways, on a rented RTX 4090 (Vast.ai, ~$0.38/hr):

1. **Naive HuggingFace `generate()`** — the unoptimized baseline. No continuous batching, no special memory management. Just a straightforward forward-pass loop.
2. **vLLM at FP16** — same numeric precision as the baseline, but served through vLLM, which adds continuous batching and PagedAttention (a more efficient way of managing KV cache memory, borrowing ideas from OS-style virtual memory paging).
3. **vLLM at FP8** — same serving engine as #2, but with a pre-quantized FP8 checkpoint instead of full-precision weights.

For each, I measured time-to-first-token (TTFT, dominated by prefill), inter-token latency (ITL, dominated by decode), total throughput, and peak GPU memory, across a fixed set of six prompts ranging from 8 to 37 tokens.

---

## Getting There Was Half the Project

Before any benchmarking happened, I burned real time on infrastructure friction. Turned out to be its own useful lesson in platform thinking.

**Disk space.** My first two rented instances had 24GB of disk — nowhere near enough for two ~16-18GB model checkpoints plus a serving environment. The fix wasn't in my code at all. It was a "Container Size" field on the instance configuration screen that I'd scrolled past. A reminder that in infra work, the bug is sometimes two abstraction layers away from where you're looking.

**A stray process squatting on the GPU.** Midway through, a leftover vLLM process from an earlier run was holding 43GB of VRAM, throwing an out-of-memory error that had nothing to do with my model or my script. `nvidia-smi` plus `kill -9` fixed it. But it's a good reminder that "the code is wrong" and "the environment is dirty" produce identical symptoms — and you have to check both before you trust either.

None of this shows up in the final numbers. It's still a fair chunk of what "hands-on" actually means in this line of work.

---

## The Results

| Method | TTFT | ITL | Throughput |
|---|---|---|---|
| Naive HF | 0.030s | 0.024s | 41 tok/s |
| vLLM FP16 | 0.020s | 0.016s | 61 tok/s |
| vLLM FP8 | 0.015s | 0.011s | 91 tok/s |

FP8 more than doubled throughput over the naive baseline — 91 vs 41 tok/s — and beat vLLM at FP16 by roughly 1.5x.

But the number I actually care about most is ITL: 0.024s → 0.016s → 0.011s.

That's the metric that isolates the memory-bound decode phase — which is where the workloads I actually care about (agent loops, multi-turn, KV-cache-heavy) spend most of their time. Not one-shot completions.

---

## The Bug That Mattered More Than the Benchmark

My first pass through this produced a result that shouldn't have been possible: FP8 came out *slower* to first token than FP16. 0.072s vs 0.024s, averaged across prompts.

That directly contradicts the mental model above. FP8 should help prefill too, via faster tensor-core throughput — not hurt it.

I didn't trust the average. I broke it down per-prompt instead.

```
prompt_len=   8  ttft=0.3542
prompt_len=  12  ttft=0.0152
prompt_len=   9  ttft=0.0140
prompt_len=  14  ttft=0.0148
prompt_len=  12  ttft=0.0159
prompt_len=  37  ttft=0.0158
```

The first prompt's TTFT was 0.354s. Roughly 20-25x every other prompt in the same run, which all landed around 0.015s.

That's not a real result. It's a one-time cost — kernel compilation, first-call memory allocation — getting misattributed entirely to whichever prompt happened to go first.

The fix was a single warmup call: generate a few throwaway tokens right after loading the model, discard the result, *then* start the timed loop.

Standard benchmarking practice. I found out why it's standard by skipping it and getting a wrong answer.

There was a second, quieter bug. My initial memory measurement (`torch.cuda.max_memory_allocated()`) reported 0 MB for both vLLM runs. vLLM manages its own memory pool somewhat independently of PyTorch's normal allocator, so PyTorch's tracking was blind to it. Reading actual GPU memory via `nvidia-smi` directly — instead of trusting a framework-specific counter — fixed it. 45GB reported instead of 0.

> When a result contradicts your mental model, that's a prompt to audit your *measurement*, not your *theory*, first.

In both cases, the theory was correct and the instrumentation was wrong. That's a healthier bug to have than the reverse. But only if you notice it instead of writing up the wrong number.

---

## Why the Corrected Results Make Sense

Once the warmup call was in place, the numbers lined up with the mental model cleanly.

Prefill is compute-bound, so FP8's faster tensor-core arithmetic wins there. Decode is memory-bound, so FP8's smaller weight footprint wins there too — for a completely different, unrelated reason. Two mechanisms, one precision change, both bottlenecks addressed. That's why FP8 won on TTFT *and* ITL, not just one.

The vLLM-vs-naive-HF gap, even at matched FP16 precision, is a separate story entirely: vLLM's serving-specific optimizations — continuous batching, PagedAttention's memory management — beat a naive forward-pass loop regardless of numeric precision.

---

## What This Means for the Work I Actually Want to Do

I'm not the person deciding *whether* to quantize a given model. That's a model or eval team's call, resting on task-specific accuracy benchmarks I didn't run here.

What I *am* responsible for, as a platform engineer serving these systems in production: given a quantization decision someone else made, how many concurrent sessions fit in a fixed GPU memory budget, what's the cost-per-token, what's the latency profile under real load — and whether a vendor's serving claims ("we run FP8 with per-channel quantization") are a reasonable default or something to be more skeptical of.

This benchmark is a small, concrete first step toward answering those questions with real numbers. Not vendor claims. Not my own assumptions before I checked.

---

## Bottom Line

The headline result — FP8 wins on both prefill and decode, for two unrelated hardware reasons — is the part I came in looking for.

The part I didn't come in looking for: a 0.354s TTFT on the first prompt of a run isn't a data point. It's an artifact. And a memory counter reading 0 MB isn't "vLLM uses no memory" — it's "you're reading the wrong counter."

Neither of those would've shown up if I'd trusted the first average and moved on.

Good benchmarks aren't about running the code once and reporting what comes out. They're about being suspicious of your own numbers until you've earned the right not to be.

---

Next up: a RAG system with a rigorous evaluation harness. Less about the pipeline, more about measuring retrieval quality honestly.
