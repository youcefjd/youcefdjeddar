---
layout: post
title: "Day 2: Measuring FP8 Quantization on vLLM"
date: 2026-07-17
description: "I predicted FP8 would roughly double decode speed. It didn't, and the reason is more interesting than the number. There's also a null result that contradicts something people often repeat about quantized reasoning models."
---

[Last time]({% link _posts/2026-07-15-standing-up-the-inference-seam.md %}), I put a GPU behind an OpenAI-compatible endpoint, measured a baseline before touching any tuning knobs, and ended on a prediction written down before running anything: FP8 should roughly halve decode latency and barely touch TTFT, because decode is bandwidth-bound and prefill is compute-bound.

Today I tested that with three experiments: a context-length sweep in bf16, the same sweep in FP8, and a quality grid to check I hadn't traded accuracy for speed without noticing.

Everything today runs at concurrency 1. Day one varied concurrency and today varies prompt length. Mixing the two would make a compute win impossible to tell apart from a queueing effect, so I kept it to one variable.

Only two of my eleven pre-registered predictions were right. That sounds bad, but the goal wasn't to be right, it was to find out where my mental model was off, and nine misses told me a lot.

---

## What I Ran

**Sweep A**: bf16, input length ∈ {128, 512, 1024, 4096, 8192} tokens, output pinned at 128, concurrency 1.

**Sweep B**: identical grid, same model at FP8.

**A memory probe**: vLLM's allocator report at both precisions, to see how much VRAM each one actually frees up.

**A quality 2×2**: 100 GSM8K math problems per cell, crossing {bf16, FP8} × {thinking on, thinking off}. I didn't have this on day one, and the speed numbers don't mean much without it.

---

## Sweep A: Where Prefill Stops Being Free

| Input tokens | TTFT (ms) | ITL (ms) | tok/s/user |
|---|---|---|---|
| 128 | 20.56 | 10.49 | 95.33 |
| 512 | 37.14 | 10.55 | 94.77 |
| 1024 | 59.43 | 10.59 | 94.43 |
| 4096 | 343.42 | 10.92 | 91.56 |
| 8192 | 727.17 | 11.26 | 88.77 |

The first thing I did with this table was check whether it adds up. Total latency grew by 805 ms from the 128-token row to the 8192-token row. TTFT alone grew by 706.6 ms. The other 127 decode steps, at the measured ITL delta, account for another 97.8 ms. That's 804.4 ms against a measured 805.0 ms, within 0.6 ms.

So "prefill accounts for the long-context cost" isn't just a story, the arithmetic backs it up. Nothing else is on the critical path, and everything below relies on that.

The more interesting thing is the *shape* of TTFT growth. I'd expected flat-then-linear: small prompts dominated by fixed overhead, a knee, then a clean linear compute cost past it. What I got instead:

| Step | Δ TTFT | ms per 1k input tokens |
|---|---|---|
| 128 → 512 | +16.6 | 43 |
| 512 → 1024 | +22.3 | 44 |
| 1024 → 4096 | +284.0 | 92 |
| 4096 → 8192 | +383.8 | 94 |

It's linear in both regions, with per-token cost roughly doubling somewhere between 1k and 4k. So it's a step between two linear regimes rather than a knee caused by overhead. I had two candidate explanations: vLLM's chunked prefill splitting long prompts at some fixed chunk boundary and paying scheduling overhead per chunk past it, or the GPU simply not saturated below ~1k tokens, so early tokens ride on idle compute that later tokens don't get.

Sweep B told them apart (details below). It was compute saturation, and I only know that because changing precision changed the slope.

There's also no quadratic blowup at the top end: 92 ms/1k from 1k→4k and 94 ms/1k from 4k→8k are practically identical. If attention's quadratic cost dominated prefill in this range, that last step would visibly steepen, and on this model at this context length it doesn't. FlashAttention is doing its job. That's useful for capacity planning: the quadratic wall is further out than 8k on an 8B model.

---

## Sweep B: FP8 Helps Both Phases

| Input | TTFT bf16 | TTFT FP8 | Δ | ITL bf16 | ITL FP8 | Δ |
|---|---|---|---|---|---|---|
| 128 | 20.56 | 15.32 | −25% | 10.49 | 7.21 | −31% |
| 512 | 37.14 | 27.83 | −25% | 10.55 | 7.22 | −32% |
| 1024 | 59.43 | 32.28 | −46% | 10.59 | 7.24 | −32% |
| 4096 | 343.42 | 211.02 | −39% | 10.92 | 7.57 | −31% |
| 8192 | 727.17 | 479.10 | −34% | 11.26 | 7.92 | −30% |

I need to correct something I said confidently on day one. I framed quantization as something that helps memory-bound work, as if that were the whole picture, and it isn't. **Both** phases improved, for two unrelated reasons. ITL drops because there are fewer bytes to stream per token, which is the bandwidth story I already had. But TTFT also drops a lot, and TTFT is prefill, which is compute-bound. That gain comes from FP8 tensor cores doing the matrix math roughly twice as fast, because this checkpoint quantizes activations as well as weights (`activation_scheme: dynamic` in the config). I checked that before running anything, since a weights-only checkpoint would have improved ITL and done nothing for TTFT.

So the updated version of day one's claim is that quantization trades accuracy for bandwidth *and* compute, and which one you notice depends on which limit your workload hits.

This also settles the chunking-vs-saturation question from Sweep A. The steep-region slope dropped from ~92-94 ms/1k to 58-65 ms/1k under FP8, a 31-37% cut. Chunking overhead is a scheduling cost and precision can't change it, while compute cost does respond to precision. So the step in Sweep A was compute saturation.

---

## The Number I Actually Care About: 1.45×, Not 2×

This is where the day got interesting.

I predicted ITL would go from 10.49 ms to about 5.25 ms, a clean halving, because FP8 halves the bytes read per token. It landed at 7.21 ms. That's a real improvement, but not the number I expected, and the gap is what I learned the most from.

Working backward from the measured latency: 7.21 ms × 1792 GB/s (this card's memory bandwidth) is about 12.9 GB read per decode token, not the ~8.2 GB you'd get from halving 16.4 GB of bf16 weights. So roughly 4.7 GB of *other* traffic comes along with every token.

That extra traffic is quantization bookkeeping. This checkpoint uses block-wise FP8, with a separate scale factor per 128×128 tile of weights stored next to the quantized values, plus activation scales computed at runtime. Every matmul has to read the weights *and* the scales, then dequantize before multiplying. bf16 doesn't pay any of this, since full precision needs no scale factors.

The memory probe backs this up independently of the latency math. FP8's KV-cache memory pool was 17.4 GiB against bf16's 10.93 GiB, a 6.47 GiB difference, when a clean halving of a 16.4 GB model should free 8.2 GiB. That implies a resident FP8 weight footprint of about 9.9 GiB, not 8.2, because the scale factors sit in VRAM the whole time.

So two different measurements, one of streamed bytes per token (from latency) and one of resident memory (from the allocator), tell the same story. They don't match each other numerically (12.9 GB streamed vs 9.9 GiB resident), and they shouldn't, because streamed traffic includes activation scales and dequant reads that never stay in VRAM, while the resident number is only the stored weights and scales. Both point at the same overhead.

> FP8 halved the weights. It did not halve what decode actually reads.

The scale/dequant overhead is roughly fixed, while the payload keeps shrinking as precision drops. So FP4 won't halve 12.9 GB again, and each further step down in precision should give smaller returns. That's a specific prediction I can test when I get to FP4.

---

## The KV Cache Prediction I Over-Called, and Why

I'd also predicted FP8 would degrade more than bf16 across context length. The KV cache stays at full precision in both cases (it isn't part of weight quantization), so as the weight term shrinks, the KV cache's fixed cost becomes a *bigger fraction* of the total. I called roughly +13-15% degradation for FP8 versus bf16's own +7.3%.

The actual FP8 number was +9.8%: right direction, wrong size. The same arithmetic mistake explains the miss. I derived 13-15% assuming FP8 reads 8.2 GB per token, but it's 12.9 GB, as shown above. With the right denominator:

```
bf16:  1.2 GB KV / 16.4 GB total = 7.3%  → measured +7.3%
FP8:   1.2 GB KV / 12.9 GB total = 9.3%  → measured +9.8%
```

With the corrected input, the prediction lands almost exactly. One wrong number early on spread into later predictions too: the ITL call and this one both came from the same bad assumption about what FP8 reads. It's worth getting the basic numbers right before building predictions on them.

---

## The Composite Metric Is Misleading

A quick methodology note, because I nearly fell for this myself. The single "output token throughput" number genai-perf reports dropped from 94.61 tok/s at 128 tokens of context to 59.32 tok/s at 8192, which reads like decode collapsed 37% under long context.

It didn't. Per-user decode throughput went from 95.33 to 88.77 tok/s, a real but much smaller 7% drop that matches the ITL numbers above. The composite metric folds prefill latency into what looks like a decode number, and at long context, prefill dominates the request's wall-clock time, so the composite tanks for reasons that have nothing to do with per-token generation speed. It's even more misleading under FP8: 137.44 down to 86.18, a 37% drop, while the real per-user number only moves 9%.

TTFT and ITL are the columns to trust. A single "tokens per second" number compared across context lengths is mostly measuring prefill and calling it throughput.

---

## 2.3×, and Who Actually Gets It

Add up what FP8 actually bought, properly attributed:

- **Per-user throughput:** +45% (95.3 → 138.7 tok/s)
- **Concurrency ceiling** (from the memory probe — more headroom for KV cache means more concurrent sequences fit): 1.59× (79,584 → 126,672 KV cache tokens)

Multiply those and you get roughly 2.3× aggregate serving capacity. "31% faster" undersells it, and "2×" doesn't show up anywhere in today's numbers.

The split from day one shows up again here. For a fleet of agents doing batch work, both gains matter and multiply: more tokens per sequence *and* more sequences in memory at once. For me, one person in a chat session, only the 45% is visible, because I'm one sequence and the concurrency ceiling doesn't affect me.

So the same optimization is worth very different amounts depending on who's using it. This is the second time a measurement has shown that, and I trust it more now that it's come up twice.

It also closes a question from day one's TTFT spread: at 128-token context, the KV pool holds space for roughly 989 concurrent sequences. Day one's heaviest run used 64 of them, 6% of capacity, so KV memory was never the bottleneck there. The TTFT blowup under concurrency was prefill compute contention, as today's results would predict, and not the memory pressure I'd been half-suspecting.

---

## The Claim I Repeated Twice Today That Turned Out to Be Wrong

Going into today, my notes said (and I'd repeated it to myself without checking) that quantization noise compounds through long reasoning chains: more generated tokens means more chances for a small numerical error to snowball into a wrong final answer.

The 2×2 doesn't support that.

| | bf16 | FP8 | Δ |
|---|---|---|---|
| Thinking on (2421 tok avg) | 95.0% | 96.0% | +1.0 |
| Thinking off (265 tok avg) | 90.0% | 91.0% | +1.0 |

The gap is identical whether the chain is 265 tokens or 2421. If quantization noise compounded with chain length, the 9x-longer reasoning trace should show a bigger FP8 gap than the short one, and it doesn't.

The +1 point isn't a real signal anyway. With n=100 the binomial standard error is around 2.2 points, so a 1-point gap is noise. The right reading is "no detectable difference," not "FP8 is slightly better." Calling +1 an improvement would be the same mistake as the composite-throughput number above.

To be precise about the scope of this null result: one model, one task, one quantization scheme, n=100 per cell. It says nothing about FP4, where the mechanism for accuracy loss might genuinely be different and worse. But at FP8, this data doesn't support the common advice that you shouldn't quantize reasoning models because the errors compound.

I'd repeated that line to other people, confidently, because it felt true the way things do after you've read them three times. At least here it wasn't, and I only found out by running the 2×2.

---

## The Thinking Tax, Priced

On day one, Qwen3 spent about 108 tokens deciding how to respond to "hi," which measured the *cost* of thinking mode. Today's quality grid measures its *value*: on GSM8K, turning thinking on is worth +5.0 accuracy points at both precisions (95.0% vs 90.0% at bf16, 96.0% vs 91.0% at FP8). That comes from the task, and quantization doesn't change it.

So now there's a real price for a routing rule. Nine times the tokens for five points of accuracy is a good deal on a hard math problem and a terrible one on "hi." The router has to tell those apart, which means routing on what kind of task it is, not just on model size or a blanket thinking-mode default.

---

## The Scorecard: 2 Out of 11

Every prediction, written down before running anything:

- TTFT shape (flat then linear) — **wrong**, it was linear-then-steeper-linear
- ITL at 128 tokens (~5.8-6.5 ms) — **wrong**, landed at 7.21
- ITL at 8192 tokens (~7.0-7.6 ms) — **right**, 7.92 close enough given the same root cause as above
- TTFT at 128 (~19-21, expected flat) — **wrong direction**, FP8 actually improved it to 15.32
- TTFT at 8192 (~450-550) — **right**, landed at 479
- KV drift under FP8 (+13-15%) — **wrong magnitude**, actual +9.8%, right direction
- KV pool ratio (~1.9×) — **wrong**, actual 1.59×
- FP8 quality gap under thinking (1-3 pts worse) — **wrong**, no detectable gap
- bf16 thinking accuracy (88-92%) — **wrong**, actual 95.0%
- bf16 no-thinking accuracy (78-85%) — **wrong**, actual 90.0%
- Average thinking tokens (600-900) — **wrong by 3×**, actual 2421

Two right. The nine misses come down to three causes, though. First, I assumed FP8 weights came to 8.2 GB, and that one wrong number broke the ITL, KV-drift and pool-ratio predictions at the same time. Second, I under-called the thinking tax by 3x even though I'd measured a 10x multiplier on a trivial prompt on day one. Knowing a number and actually using it in a forecast turn out to be different things. Third, I believed the compounding-noise claim without testing it, and it didn't hold up in a 2×2 with a control.

Day one had nothing to predict against. Day two had eleven predictions on record and got two right, which is a better result than it sounds. If a benchmark never surprises you, it's only confirming what you already believed.

---

## The Bug That Mattered More Than Any Number Today

Before trusting any of the quality-grid numbers above, I had to throw out an earlier run of the same harness. The first pass gave 71% accuracy on GSM8K, which looked respectable enough to publish, and it was wrong.

I'd capped `max_tokens` at 2048 for the quality harness, which I thought was generous for math problems. Average completion length came out to 2421 tokens. The distribution was jammed against a ceiling I'd called generous, and a huge share of responses got cut off mid-reasoning-trace before reaching a final answer.

The part that should have set off alarms, and almost didn't: my scoring regex was grabbing the last number-looking thing out of a truncated, unfinished reasoning trace and sometimes scoring it correct anyway, purely by chance. 43 of the runs were truncated, and across the 57 completed runs plus those 43, I still got 71 "correct" answers. That's only possible if unfinished traces were sometimes landing on the right number by accident. So some real failures were counted as passes, and the total still looked like a perfectly ordinary accuracy score.

Raising `max_tokens` to 8192 and re-running moved the number from 71% to 95%. That's 24 points of measurement artifact, with no visible sign anything was wrong. I now check `finish_reason: length` before believing any accuracy number from this harness. If any completion hit the token cap instead of stopping naturally, the run doesn't count until the cap is fixed and it's rerun.

Day one's lesson was to measure before tuning. Today's is to check the measuring tool before believing what it tells you. A broken eval doesn't warn you, it just gives you a believable number.

---

## Where This Leaves the Frontier Table

Two precisions, five context lengths, a memory ratio, a quality grid with a control built into it. That might already be enough to call Band 1 of this project done. The table is useful as it is, and the bigger risk now is "one more backend" turning a three-week budget into an eight-week one.

The options for next time: push into FP4, the more speculative bet, testing whether the scale/dequant tax found today collapses the returns the way I think it will; run the same grid through SGLang instead of vLLM, the actual apples-to-apples serving-stack comparison this platform needs; or stop here and ship the table, because it might already be the deliverable.

One thing I owe myself before any of that: `vllm`'s `/metrics` endpoint has been sitting at `localhost:8000/metrics` this entire time, and I've been inferring queue depth from TTFT spread when `num_requests_running` and `num_requests_waiting` were one `curl` away all day. That's tomorrow's first command, regardless of which path I pick.

I spent a whole day inferring something I could have just looked up. I'll check first next time.
