---
layout: post
title: "What I Learned Benchmarking Llama 3 8B Three Ways"
date: 2026-07-10
description: "The results were satisfying. The bug I hit along the way taught me more."
---

I wanted to actually understand how model serving works, not just repeat facts I'd read in a blog post.

So this week I benchmarked the same 8B-parameter model served three different ways, and tried to work out why the numbers come out the way they do. The results were satisfying, but a bug I hit along the way taught me more.

---

## Why This Project, and Why Now

Before touching any code, I spent time building a mental model of how LLM inference actually works, mechanically: what a KV cache is, why inference splits into two phases with different performance characteristics, and what quantization is actually doing to the numbers under the hood.

I wanted the vocabulary and intuition *before* deploying anything. Otherwise "FP8 is faster" is just a fact I memorized, and I can't reason with it when a real system misbehaves. That turned out to matter a lot by the end of the project.

---

## The Mental Model, Briefly

Every LLM inference request has two phases with very different performance profiles.

**Prefill**: the model processes your entire input prompt at once, all tokens, in parallel, through every layer. This is **compute-bound**: the GPU's tensor cores are the bottleneck, not memory.

**Decode**: the model generates output one token at a time, sequentially. Each step reloads the same weight matrices from GPU memory to do a comparatively tiny amount of math. This is **memory-bound**: the compute cores mostly sit idle, waiting for data to show up.

Every generated token also produces a Key and Value vector, cached in GPU memory so future tokens can attend back to it without recomputing it from scratch. That KV cache grows by one entry per token, per layer. For a 32-layer model generating a long response that's a lot of accumulated state, and it's a big part of why long conversations get progressively more expensive to serve.

Quantization (dropping weights from 16-bit floating point to 8-bit) helps *both* phases, but for two unrelated hardware reasons:

- It helps **prefill** because GPU tensor cores are physically faster at FP8 arithmetic than FP16, with simpler circuitry and more operations per clock cycle. That's about compute throughput.
- It helps **decode** because FP8 weights are half the bytes of FP16 weights, so there's half as much data to move from memory to the compute cores on every token generated. That's about data transfer.

So one precision change addresses both bottlenecks through two separate mechanisms. That's what I wanted to see in real numbers instead of taking a blog post's word for it.

---

## The Setup

I served Llama 3 8B Instruct three ways, on a rented RTX 4090 (Vast.ai, ~$0.38/hr):

1. **Naive HuggingFace `generate()`**: the unoptimized baseline, with no continuous batching or special memory management, just a straightforward forward-pass loop.
2. **vLLM at FP16**: same numeric precision as the baseline, but served through vLLM, which adds continuous batching and PagedAttention (a more efficient way of managing KV cache memory, borrowing ideas from OS-style virtual memory paging).
3. **vLLM at FP8**: same serving engine as #2, but with a pre-quantized FP8 checkpoint instead of full-precision weights.

For each, I measured time-to-first-token (TTFT, dominated by prefill), inter-token latency (ITL, dominated by decode), total throughput, and peak GPU memory, across a fixed set of six prompts ranging from 8 to 37 tokens.

---

## Getting There Was Half the Project

Before any benchmarking happened, I lost a good amount of time to infrastructure friction, which ended up being a useful lesson of its own.

**Disk space.** My first two rented instances had 24GB of disk, nowhere near enough for two ~16-18GB model checkpoints plus a serving environment. The fix wasn't in my code at all. It was a "Container Size" field on the instance configuration screen that I'd scrolled past. In infra work the bug is sometimes two abstraction layers away from where you're looking.

**A stray process squatting on the GPU.** Midway through, a leftover vLLM process from an earlier run was holding 43GB of VRAM, throwing an out-of-memory error that had nothing to do with my model or my script. `nvidia-smi` plus `kill -9` fixed it, but "the code is wrong" and "the environment is dirty" produce identical symptoms, and you have to check both.

None of this shows up in the final numbers, but it's a fair chunk of what hands-on work actually looks like.

---

## The Results

| Method | TTFT | ITL | Throughput |
|---|---|---|---|
| Naive HF | 0.030s | 0.024s | 41 tok/s |
| vLLM FP16 | 0.020s | 0.016s | 61 tok/s |
| vLLM FP8 | 0.015s | 0.011s | 91 tok/s |

FP8 more than doubled throughput over the naive baseline (91 vs 41 tok/s) and beat vLLM at FP16 by roughly 1.5x.

The number I care about most is ITL: 0.024s → 0.016s → 0.011s. It isolates the memory-bound decode phase, which is where the workloads I care about (agent loops, multi-turn, KV-cache-heavy) spend most of their time, much more than one-shot completions do.

---

## The Bug That Mattered More Than the Benchmark

My first pass through this produced a result that shouldn't have been possible: FP8 came out *slower* to first token than FP16. 0.072s vs 0.024s, averaged across prompts.

That contradicts the mental model above, since FP8 should help prefill through faster tensor-core throughput. I didn't trust the average, so I broke it down per prompt.

```
prompt_len=   8  ttft=0.3542
prompt_len=  12  ttft=0.0152
prompt_len=   9  ttft=0.0140
prompt_len=  14  ttft=0.0148
prompt_len=  12  ttft=0.0159
prompt_len=  37  ttft=0.0158
```

The first prompt's TTFT was 0.354s, roughly 20-25x every other prompt in the same run, which all landed around 0.015s. That isn't a real result. It's a one-time cost (kernel compilation, first-call memory allocation) being charged entirely to whichever prompt happened to go first.

The fix was a single warmup call: generate a few throwaway tokens right after loading the model, discard them, and only then start the timed loop. It's standard benchmarking practice, and I found out why by skipping it and getting the wrong answer.

There was a second, quieter bug. My initial memory measurement (`torch.cuda.max_memory_allocated()`) reported 0 MB for both vLLM runs. vLLM manages its own memory pool somewhat independently of PyTorch's normal allocator, so PyTorch's tracking was blind to it. Reading GPU memory directly with `nvidia-smi` instead of trusting a framework-specific counter fixed it: 45GB instead of 0.

> When a result contradicts your mental model, that's a prompt to audit your *measurement*, not your *theory*, first.

In both cases the theory was right and the instrumentation was wrong. That's a better kind of bug to have than the reverse, as long as you notice it before writing up the wrong number.

---

## Why the Corrected Results Make Sense

With the warmup call in place, the numbers lined up with the mental model.

Prefill is compute-bound, so FP8's faster tensor-core arithmetic helps there. Decode is memory-bound, so FP8's smaller weights help there too, for an unrelated reason. That's why FP8 won on both TTFT *and* ITL.

The gap between vLLM and naive HF at the same FP16 precision is a separate story. vLLM's serving optimizations (continuous batching, PagedAttention's memory management) beat a naive forward-pass loop regardless of precision.

---

## Finding the Actual Capacity Ceiling

The microbenchmark above tells you which serving method is faster. It doesn't tell you what matters for capacity planning: how many concurrent requests one GPU can take before it falls over.

So I extended it. I ran vLLM in server mode (instead of the offline batch API I'd been using) and fired concurrent requests at it via asyncio/aiohttp, ramping concurrency from 32 up to 512, against Llama 3 8B at FP16 on the same RTX 4090.

- **32 → 64 concurrent requests**: throughput scales close to linearly, 17 → 31 req/s, and latency barely moves.
- **64 → 128**: this is where it breaks. Latency jumps and throughput stops keeping up.
- **128 and up**: throughput plateaus around ~38 req/s. Extra concurrent requests just queue.

You can tell it's a queueing problem rather than a compute problem because the gap between average and max latency keeps widening past 128. If the GPU itself were the bottleneck, average and max would move together and everyone would be equally slow. Instead some requests get through quickly while others wait in a growing line, which points at scheduling.

I'm not the one deciding *whether* to quantize a given model. That's the model or eval team's call, based on accuracy benchmarks I didn't run here. But this is the number I need for my part of the job: on this hardware and precision, ~64 concurrent requests is where the free throughput runs out, and ~38 req/s is the ceiling however many clients you add. That's a capacity budget I can plan around instead of guessing.

---

## Bottom Line

The headline result, FP8 winning on both prefill and decode for two unrelated hardware reasons, is what I came looking for.

What I didn't expect to learn: a 0.354s TTFT on the first prompt of a run was an artifact, and a memory counter reading 0 MB meant I was reading the wrong counter, not that vLLM uses no memory. I'd have missed both if I'd trusted the first average and moved on.

The concurrency test isn't a production-readiness claim either. It's one GPU, one model, one precision and synthetic load. It does show I can take a serving stack and come back with a real capacity number, which is the skill I was practicing this week.

The main lesson is to stay suspicious of your own numbers until you've checked how they were measured.

---

Next up is a RAG system with a proper evaluation harness, focused more on measuring retrieval quality honestly than on the pipeline itself.
