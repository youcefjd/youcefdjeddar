---
layout: post
title: "A Museum of Inefficiencies: Why I Stopped Trying to Beat Prediction Markets"
date: 2026-02-06
description: "I found a 13% edge in Polymarket. I'm not going to trade it."
---

You've seen the tweets.

"I used Claude to build a trading bot that makes me $3,000/day."

"AI + prediction markets = passive income on autopilot."

"Just vibe code your way to financial freedom."

I wanted to test this empirically. Not with vibes. With data.

I found a 13% edge in Polymarket.

I'm not going to trade it.

Let me explain.

---

## The Dream

Like every engineer who's ever stared at a terminal and wondered if it could print money, I wanted passive income.

Not "start a business" passive income. Not "build an audience" passive income. The real kind: find a market inefficiency, automate it, compound returns, and retire to a beach somewhere while my bots print money.

Prediction markets seemed perfect.

Polymarket. Kalshi. Platforms where people bet on real-world events—sports, crypto, politics. Surely someone is mispricing something.

So I did what any reasonable person would do:

I built five daemons to monitor 1,600+ markets over three days, tested four statistical hypotheses with locked-in significance thresholds, and paper-traded every signal that looked promising.

You know. Normal Tuesday stuff.

---

## The Setup

Here's what I was tracking:

- **Short-horizon** — Are favorites/underdogs mispriced in <48h markets?
- **Flow tracker** — Can I follow whale money and ride the wave?
- **New market tracker** — Is there a "stale window" where new markets are mispriced?
- **Player prop tracker** — Are NBA player props systematically wrong?

Each hypothesis had a clear threshold:

- \>10% edge
- Statistically significant (p < 0.05)
- After transaction costs

No peeking. No mid-experiment adjustments. Lock the criteria and let the data speak.

This is how you avoid fooling yourself.

(I've read enough quant Twitter to know most "edges" are just overfitting in a hoodie.)

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

Here's the part that matters:

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

Because I didn't — and now I know empirically, not theoretically.

Most people lose money discovering that markets are efficient.

I spent three days and $0 to reach the same conclusion.

I also got a blog post out of it.

So really, I'm up.

---

## The Takeaway

If you're thinking about trying to beat prediction markets:

- You probably can't
- The real edges are either too small (costs) or too illiquid (spreads)
- This isn't a bug — it's the market working correctly

If you find a "free money" opportunity, ask why it still exists.

The answer is usually: because it isn't.

I wanted passive income from market inefficiencies.

I got a lesson in market efficiency instead.

Fair trade.

Now if you'll excuse me, I need to go find another way to not make money.
