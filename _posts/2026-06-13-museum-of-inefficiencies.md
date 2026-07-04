---
layout: post
title: "A Museum of Inefficiencies: What Four Killed Trading Signals Taught Me About Discipline"
date: 2026-06-13
description: "Four hypotheses, locked significance thresholds, and the real reason none of them were tradeable — a case study in the discipline that separates a real edge from noise."
---

You've seen the tweets.

"I used Claude to build a trading bot that makes me $3,000/day."

"AI + prediction markets = passive income on autopilot."

"Just vibe code your way to financial freedom."

I wanted to test the underlying claim empirically. Not with vibes. With data — and with the same discipline I'd want from anyone telling me they'd found an edge.

I found a 13% edge in Polymarket.

I'm not going to trade it.

Let me explain why, and why the "why" matters more than the number.

---

## The Question

The actual question wasn't "can I get rich." It was simpler and more useful: **can I tell a real, tradeable edge apart from noise, before I've put a dollar behind it?**

Most "edges" people report — in prediction markets, in any market — are overfitting wearing a disguise. The only way to know if mine was different was to lock the rules before looking at the results, and be willing to walk away from a number that looked good but didn't survive scrutiny.

So I did the boring, correct version of "find alpha":

I built five daemons to monitor 1,600+ markets over three days, tested four statistical hypotheses with locked-in significance thresholds, and paper-traded every signal that looked promising.

---

## The Setup

Here's what I was tracking:

- **Short-horizon** — Are favorites/underdogs mispriced in <48h markets?
- **Flow tracker** — Can I follow whale money and ride the wave?
- **New market tracker** — Is there a "stale window" where new markets are mispriced?
- **Player prop tracker** — Are NBA player props systematically wrong?

Each hypothesis had a clear threshold, fixed before I collected a single data point:

- \>10% edge
- Statistically significant (p < 0.05)
- After transaction costs

No peeking. No mid-experiment adjustments. Lock the criteria and let the data speak.

This is the whole game. Everything downstream of this decision is just bookkeeping.

---

## Hypothesis 1: Underdog Calibration

**The idea:** Markets price underdogs at 30%, but they actually win 40%. Free money.

**What happened:**

- Day 0.5: +18% edge (n=31) — holy shit
- Day 1: +12% edge (n=91) — hmm
- Day 2: +11% edge (n=165) — okay
- Day 3: +7% edge (n=200+) — there it is

Classic regression to the mean.

By the time I had enough data to trust the signal, it had decayed to +7% — which disappears entirely after Polymarket's ~5% transaction costs.

**Verdict: Dead.**

---

## Hypothesis 2: Flow Following

**The idea:** Forget predicting outcomes. Detect when smart money moves and ride the wave.

Built a tracker for:

- Volume spikes
- Price velocity
- Whale-sized entries

Monitored 300+ markets.

**Result:** +6.5% edge. Stable. Consistent.

Also: exactly what transaction costs eat.

The edge exists. You just can't keep any of it.

**Verdict: Dead.**

---

## Hypothesis 3: The Stale Window

**The idea:** New markets launch with naive prices. Smart money takes hours to correct them. Bet early, capture correction.

**What I found:**

- Sports markets show ~43% initial pricing error
- Most correction happens in the 6–24h window

This is real. Measurable. Encouraging.

Also completely random in direction.

Sometimes initial price is too high. Sometimes too low. No consistent bias. No signal. Just large noise.

**Verdict: Dead.**

---

## Hypothesis 4: Player Prop OVERs

**The idea:** NBA player props are created by amateurs. OVERs are systematically underpriced.

**What I found:**

- **Points O/U** — +7.5% bias, 44% win rate
- **Assists O/U** — +13.6% bias, 49% win rate
- **Rebounds O/U** — +11.7% bias, 47% win rate

Wait.

This is real.

Assists and rebounds show consistent 11–14% mispricing. Statistically significant. Across 200+ markets. Correction happens in the 3–6h window.

I found an edge.

I should be celebrating.

I am not celebrating.

---

## The Twist

Here's the paper-trading reality:

- **Trades opened:** 8
- **Avg liquidity:** $17
- **Avg spread:** 84%
- **Trades with >$500 liquidity:** 0
- **Trades with <15% spread:** 1

An 84% spread means if the "price" is 50%, you're actually buying at 96% and selling at 4%.

The +13% correction I found? It happens inside a spread I can't cross.

Sample market:

> Trendon Watford Assists O/U: $1 liquidity, 96% spread

I can observe the inefficiency.

I cannot touch it.

**Verdict: Real edge. Zero dollars.**

---

## The Meta-Lesson

Here's the part that matters more than any of the four hypotheses:

> **Edges persist precisely where they cannot be traded.**

If the +13% player-prop edge were executable, arbitrageurs would have closed it years ago.

The fact that it still exists is proof that it's untradeable.

Polymarket player props are a museum of inefficiencies.

You can walk through. Observe the exhibits. Take notes.

You cannot take anything home.

The market isn't stupid. It's efficient exactly where it needs to be — at the point of execution. Everything else is theater.

---

## The Scoreboard

- **Underdog calibration** — +7% edge → No (costs) → $0
- **Flow following** — +6.5% edge → No (costs) → $0
- **Stale window** — 43% error → No (random) → $0
- **Player prop OVERs** — +13% edge → No (liquidity) → $0

3 days. 1,600 markets. 5 daemons. 4 hypotheses.

Tradeable edges found: 0.

---

## Was It Worth It?

Yes.

Not because I found alpha.

Because I didn't — and now I know empirically, not theoretically, and I know exactly which of the four failure modes killed each hypothesis: decay, cost, noise, or liquidity. That's a more useful output than a number would have been.

Most people lose money discovering that markets are efficient.

I spent three days and $0 to reach the same conclusion.

---

## The Takeaway

If you're evaluating a claimed edge — yours or someone else's — the number is the least interesting part. The questions that actually matter:

- Did the edge survive out-of-sample, or did it decay the moment you collected more data?
- Does it clear transaction costs, or does it just look good before them?
- Can you actually execute it, or does the liquidity disappear exactly where the edge appears?

If you find a "free money" opportunity, ask why it still exists.

The answer is usually: because it isn't — and the same discipline that kills a bad prediction-market signal is exactly what should be applied to any model or agent claiming it found something real, in any market.

I wanted passive income from market inefficiencies.

I got a sharper filter for spotting fake edges instead.

Fair trade.
