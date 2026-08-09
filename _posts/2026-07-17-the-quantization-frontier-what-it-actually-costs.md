---
layout: post
title: "The Quantization Frontier, and What It Actually Costs"
date: 2026-07-17
description: "I predicted FP8 would roughly double decode speed. It didn't. The reason why is more interesting than the number would've been — and it comes with a null result that contradicts something everyone repeats about quantized reasoning models."
---

[Last time]({% link _posts/2026-07-15-standing-up-the-inference-seam.md %}), I put a GPU behind an OpenAI-compatible endpoint, measured a baseline before touching any tuning knobs, and ended on a prediction written down before running anything: FP8 should roughly halve decode latency and barely touch TTFT, because decode is bandwidth-bound and prefill is compute-bound.

Today tested that. Three experiments: a context-length sweep in bf16, the identical sweep in FP8, a quality grid to check I hadn't just traded accuracy for speed without noticing.

Scope discipline that mattered: everything today runs at concurrency 1. Day one varied concurrency; today varies prompt length. Mixing the two would make a compute win indistinguishable from a queueing effect, and I only get to trust one isolated variable.

Two of eleven pre-registered predictions were right. That's the correct shape for a day that actually taught me something.

Nine wrong predictions sounds like a bad day. It was, if the goal was being right. It wasn't the goal.

---

## What I Ran

**Sweep A** — bf16, input length ∈ {128, 512, 1024, 4096, 8192} tokens, output pinned at 128, concurrency 1.

**Sweep B** — identical grid, same model at FP8.

**A memory probe** — vLLM's allocator report at both precisions, to see how much VRAM each one actually frees up.

**A quality 2×2** — 100 GSM8K math problems per cell, crossing {bf16, FP8} × {thinking on, thinking off}. The axis I didn't have on day one, and every number above it is worthless without it.

---

## Sweep A: Where Prefill Stops Being Free

| Input tokens | TTFT (ms) | ITL (ms) | tok/s/user |
|---|---|---|---|
| 128 | 20.56 | 10.49 | 95.33 |
| 512 | 37.14 | 10.55 | 94.77 |
| 1024 | 59.43 | 10.59 | 94.43 |
| 4096 | 343.42 | 10.92 | 91.56 |
| 8192 | 727.17 | 11.26 | 88.77 |

First thing I did with this table: check whether it adds up. Total latency grew by 805 ms from the 128-token row to the 8192-token row. TTFT alone grew by 706.6 ms. The other 127 decode steps, at the measured ITL delta, account for another 97.8 ms. 804.4 ms against an actual measured delta of 805.0 ms — within 0.6 ms.

That check is the difference between "prefill owns the long-context cost" being a story I'm telling and it being arithmetic. Nothing else is on the critical path. Every attribution below leans on that closed loop.

The more interesting thing is the *shape* of TTFT growth. I'd expected flat-then-linear: small prompts dominated by fixed overhead, a knee, then a clean linear compute cost past it. What I got instead:

| Step | Δ TTFT | ms per 1k input tokens |
|---|---|---|
| 128 → 512 | +16.6 | 43 |
| 512 → 1024 | +22.3 | 44 |
| 1024 → 4096 | +284.0 | 92 |
| 4096 → 8192 | +383.8 | 94 |

Linear in both regions, with per-token cost roughly doubling between the 1k and 4k mark. Not a knee from overhead — a step between two different linear regimes. Two candidate explanations: vLLM's chunked prefill splitting long prompts at some fixed chunk boundary and paying scheduling overhead per chunk past it, or the GPU simply not saturated below ~1k tokens, so early tokens ride on idle compute that later tokens don't get.

Sweep B told them apart. More below — but it was compute saturation, and I only know that because precision changed the slope.

One more thing worth naming: no quadratic blowup at the top end. 92 ms/1k from 1k→4k, 94 ms/1k from 4k→8k — practically identical. If attention's quadratic cost were dominating prefill at this range, that last step should visibly steepen. It doesn't, on this model, at this context length. FlashAttention is doing its job. A real capacity-planning number, not trivia: the quadratic wall is further out than 8k on an 8B model.

---

## Sweep B: FP8 Wins Two Fights, Not One

| Input | TTFT bf16 | TTFT FP8 | Δ | ITL bf16 | ITL FP8 | Δ |
|---|---|---|---|---|---|---|
| 128 | 20.56 | 15.32 | −25% | 10.49 | 7.21 | −31% |
| 512 | 37.14 | 27.83 | −25% | 10.55 | 7.22 | −32% |
| 1024 | 59.43 | 32.28 | −46% | 10.59 | 7.24 | −32% |
| 4096 | 343.42 | 211.02 | −39% | 10.92 | 7.57 | −31% |
| 8192 | 727.17 | 479.10 | −34% | 11.26 | 7.92 | −30% |

Time to correct something I said confidently on day one: "quantization helps memory-bound work" was my whole frame, stated as if it were the complete picture. It isn't. **Both** regimes improved, through two unrelated mechanisms. ITL drops because there are fewer bytes to stream per token — the bandwidth story I already had. But TTFT drops too, by a lot, and TTFT is prefill, and prefill is compute-bound. That win comes from FP8 tensor cores doing the actual matrix math roughly twice as fast, because this checkpoint quantizes activations too, not just weights (`activation_scheme: dynamic` in the config — checked for that before running anything, specifically because a weights-only checkpoint would've given the ITL win and nothing on TTFT).

Honest update to day one's claim: quantization trades accuracy for bandwidth *and* compute. Which one you notice depends entirely on which ceiling your workload binds against.

This also settles the chunking-vs-saturation question from Sweep A: the steep-region slope dropped from ~92-94 ms/1k to 58-65 ms/1k under FP8 — a 31-37% cut. Chunking overhead is a scheduling cost; precision can't touch scheduling. Compute cost responds to precision. So the step in Sweep A was compute saturation, confirmed by an independent lever.

---

## The Number I Actually Care About: 1.45×, Not 2×

Here's where the day got interesting instead of just confirmatory.

I predicted ITL would go from 10.49 ms to about 5.25 ms — a clean halving, because FP8 halves the bytes read per token. It landed at 7.21 ms. Real improvement, wrong number, and the gap is the finding.

Working backward from the measured latency: 7.21 ms × 1792 GB/s (this card's memory bandwidth) is about 12.9 GB read per decode token. Not the ~8.2 GB I'd have gotten from just halving 16.4 GB of bf16 weights. Something like 4.7 GB of *other* traffic rides along on every single token.

That something is quantization bookkeeping. This checkpoint uses block-wise FP8 — a separate scale factor per 128×128 tile of weights, stored alongside the quantized values, plus activation scales computed dynamically at runtime. Every matmul has to read the weights *and* the scales, then dequantize before the actual multiply. bf16 pays none of this tax; full precision needs no scale factors at all.

The memory probe backs this up independently of the latency math. FP8's KV-cache memory pool was 17.4 GiB against bf16's 10.93 GiB — a 6.47 GiB difference, when a clean halving of a 16.4 GB model should free 8.2 GiB. Implied resident FP8 weight footprint: about 9.9 GiB, not 8.2. Those scale factors sit in VRAM the whole time.

Two completely different instruments — one measuring streamed bytes per token via latency, one measuring resident memory via the allocator — land on the same story from two different angles. They don't agree with *each other* numerically (12.9 GB streamed vs 9.9 GiB resident), and they shouldn't: streamed traffic includes activation scales and dequant reads that never sit resident in VRAM, while the resident number is just the parked weights-plus-scales. Both point at the same tax.

> FP8 halved the weights. It did not halve what decode actually reads.

The scale/dequant overhead is roughly a fixed tax, while the payload it's attached to keeps shrinking as you go to lower precision. Going to FP4 next won't halve 12.9 GB again — the fixed tax stays put while the actual payload shrinks, so returns from each further step down in precision should compress. A specific, falsifiable prediction for whenever I get to FP4.

---

## The KV Cache Prediction I Over-Called, and Why

I'd also predicted FP8 would degrade more than bf16 across context length, because the KV cache stays at full precision in both cases — it isn't part of weight quantization — so as the weight term shrinks, the KV cache's fixed cost becomes a *bigger fraction* of the total. I called roughly +13-15% degradation for FP8 versus bf16's own +7.3%.

Actual FP8 number: +9.8%. Direction correct, magnitude wrong — and the day-one arithmetic mistake explains the miss exactly. I derived 13-15% assuming FP8's per-token read was 8.2 GB. It's 12.9 GB, per the section above. Redo the fraction with the right denominator:

```
bf16:  1.2 GB KV / 16.4 GB total = 7.3%  → measured +7.3%
FP8:   1.2 GB KV / 12.9 GB total = 9.3%  → measured +9.8%
```

Same mechanism, corrected input, prediction lands almost exactly. One wrong number early in the day propagated into two more predictions later — the ITL call and this one both trace back to the same bad assumption about what FP8 actually reads. Cheap lesson: one bad input contaminates everything downstream of it, so get the foundational number right before building on it.

---

## The Composite Metric That Lies, Twice

Quick methodology note, because I almost fell into it myself. The single "output token throughput" number genai-perf reports dropped from 94.61 tok/s at 128 tokens of context to 59.32 tok/s at 8192 — a headline that reads as "decode collapsed 37% under long context."

Decode did not collapse. Per-user decode throughput went from 95.33 to 88.77 tok/s — a real but much smaller 7% hit, exactly matching the ITL numbers above. The composite metric folds prefill latency into what looks like a decode number, and at long context, prefill dominates the request's wall-clock time, so the composite tanks for reasons that have nothing to do with per-token generation speed. It's worse under FP8, not better — 137.44 down to 86.18, a 37% drop, while the real per-user number only moves 9%.

TTFT and ITL are the honest columns. Anyone benchmarking "tokens per second" as a single number across different context lengths is quietly measuring prefill contamination and calling it a throughput result.

---

## 2.3×, and Who Actually Gets It

Add up what FP8 actually bought, properly attributed:

- **Per-user throughput:** +45% (95.3 → 138.7 tok/s)
- **Concurrency ceiling** (from the memory probe — more headroom for KV cache means more concurrent sequences fit): 1.59× (79,584 → 126,672 KV cache tokens)

Multiply those and you get roughly 2.3× aggregate serving capacity. Not "31% faster," which undersells it by half. Not "2×," which was never actually true anywhere in today's numbers.

The same split from day one's TTFT-vs-throughput finding shows up again, from a different angle. For a fleet of agents doing batch work, both numbers matter and compound — more tokens per sequence *and* more sequences fit in memory at once. For me, one human on a chat session, only the 45% is visible. The concurrency ceiling doesn't exist from my seat; I'm one sequence, not a hundred.

Same optimization. Genuinely different value depending on who's asking. Second time this exact structural fact has fallen out of a measurement instead of an argument I made up front, and I trust it more the second time.

It also closes a question from day one's TTFT spread: at 128-token context, the KV pool holds space for roughly 989 concurrent sequences. Day one's heaviest run used 64 of them — 6% of capacity. KV memory was never the bottleneck in that run. The TTFT blowup under concurrency was pure prefill compute contention, exactly like today's compute-bound framing predicts, not a memory-pressure problem I'd been half-suspecting.

---

## The Claim I Repeated Twice Today That Turned Out to Be Wrong

Somewhere in my notes going into today, I'd written — and repeated to myself without checking — that quantization noise compounds through long reasoning chains: more generated tokens means more chances for a small numerical error to snowball into a wrong final answer.

The 2×2 says no.

| | bf16 | FP8 | Δ |
|---|---|---|---|
| Thinking on (2421 tok avg) | 95.0% | 96.0% | +1.0 |
| Thinking off (265 tok avg) | 90.0% | 91.0% | +1.0 |

Identical gap, whether the chain is 265 tokens or 2421. If quantization noise actually compounded with chain length, the 9x-longer reasoning trace should show a bigger FP8 gap than the short one. It shows exactly the same one.

+1 point isn't a real signal to begin with — with n=100 the binomial standard error is around 2.2 points, so a 1-point gap is noise, not "FP8 is slightly better." The honest reading is "no detectable difference," not "FP8 wins." Reporting +1 as an improvement would be the same mistake as the composite-throughput number above: reading a plausible-looking number as if it were signal.

Scope of that null result, to be precise: one model, one task, one quantization scheme, n=100 per cell. It says nothing about FP4, where the mechanism for accuracy loss might genuinely be different and worse. But the version of "don't quantize reasoning models too aggressively, the errors compound" that gets repeated as received wisdom is doing work at FP8 that this data doesn't support.

I'd repeated that line to other people. Confidently. It felt true the way things you've read three times start to feel true. It just wasn't, at least not here, and the only way I found out was by actually running the 2×2 instead of continuing to nod along.

---

## The Thinking Tax, Priced

Day one, Qwen3 spent about 108 tokens deliberating over how to respond to "hi." That measured the *cost* of thinking mode. Today's quality grid measures the *value*: on GSM8K, turning thinking on is worth +5.0 accuracy points, consistently, at both precisions — 95.0% vs 90.0% at bf16, 96.0% vs 91.0% at FP8. A property of the task, not something quantization changes.

That's a router rule with an actual price tag now. Nine times the tokens for five points of accuracy is a good trade on a genuinely hard math problem. Catastrophic on "hi." Whatever router I build has to tell those two situations apart — which argues for routing decisions keyed on the *shape of the task*, not just model size or a blanket thinking-mode default.

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

Two right. But the nine misses aren't nine separate failures of intuition — they cluster into three root causes. First, I assumed FP8 weights came out to 8.2 GB, and that single wrong number broke the ITL prediction, the KV-drift prediction, and the pool-ratio prediction simultaneously — one bad input, three wrong outputs downstream. Second, I under-called the thinking tax by 3x despite having measured a 10x multiplier on a trivial prompt on day one — knowing a number and actually using it in a forecast are two different skills. Third, I believed the compounding-noise claim without ever testing it myself, and it didn't survive contact with an actual 2×2 with a control built in.

Day one had nothing to predict against, and the reconciliation came out clean. Day two had eleven predictions on the record before running anything, and got two right. That's a better day than it sounds — a benchmark that never surprises you isn't teaching you anything, it's just confirming what you already believed.

---

## The Bug That Mattered More Than Any Number Today

Before trusting any of the quality-grid numbers above, I had to throw out an earlier run of the same harness. First pass: 71% accuracy on GSM8K. Respectable. Publishable, even.

It was fiction.

I'd capped `max_tokens` at 2048 for the quality harness — generous, I thought, for math problems. Average completion length came out to 2421 tokens. The distribution was jammed against a ceiling I'd called generous, and a huge share of responses got cut off mid-reasoning-trace before reaching a final answer.

The part that should've been a five-alarm fire and almost wasn't: my scoring regex was grabbing the last number-looking thing out of a truncated, unfinished reasoning trace and sometimes scoring it correct anyway, purely by chance. 43 of the runs were truncated, and out of 57 completed runs plus those 43 truncated ones, I still got 71 "correct" answers — only possible if truncated, unfinished traces were occasionally landing on the right number by accident. Wrong in both directions at once: some real failures got miscounted as passes, and the overall number still looked like a plausible, unremarkable accuracy score you'd feel fine publishing.

Bumping `max_tokens` to 8192 and re-running moved the number from 71% to 95%. Twenty-four points of pure measurement artifact, on a benchmark that gave zero visible indication anything was wrong. `finish_reason: length` is now a hard gate I check before believing any accuracy number out of this harness — if any completion hit the token cap instead of stopping naturally, the run doesn't count until the cap is fixed and it's rerun.

Day one's lesson was measure before you tune. Today's is the same lesson one level up: check your instrument before you believe the number it just gave you. A broken eval doesn't announce itself. It hands you a plausible number and lets you walk away happy.

---

## Where This Leaves the Frontier Table

Two precisions, five context lengths, a memory ratio, a quality grid with a control built into it. That might already be enough to call Band 1 of this project done — the table is genuinely useful as-is, and the biggest risk from here isn't missing data, it's "one more backend" turning a three-week budget into an eight-week one.

The honest options for next time: push into FP4, the more speculative bet, testing whether the scale/dequant tax found today collapses the returns the way I think it will; run the same grid through SGLang instead of vLLM, the actual apples-to-apples serving-stack comparison this platform needs; or stop here and ship the table, because it might already be the deliverable.

One thing I owe myself before any of that: `vllm`'s `/metrics` endpoint has been sitting at `localhost:8000/metrics` this entire time, and I've been inferring queue depth from TTFT spread when `num_requests_running` and `num_requests_waiting` were one `curl` away all day. That's tomorrow's first command, regardless of which path I pick.

A full day spent inferring a number that was one `curl` away the whole time isn't a clever workaround. It's just not having checked. Noted, for next time.
