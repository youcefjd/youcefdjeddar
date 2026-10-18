---
layout: post
title: "A Museum of Inefficiencies: What Four Killed Trading Signals Taught Me About Discipline"
date: 2026-06-13
description: "Four hypotheses, locked significance thresholds, and the real reason none of them were tradeable — a case study in the discipline that separates a real edge from noise."
---

You've seen the tweets. "I used Claude to build a trading bot that makes me $3,000/day." "AI + prediction markets = passive income on autopilot." "Just vibe code your way to financial freedom."

I wanted to test this with actual data. I found a 13% edge on Polymarket, and I'm not going to trade it. Here's why.

---

## The Dream

Like every engineer who's ever stared at a terminal and wondered if it could print money, I wanted passive income. Not the "start a business" or "build an audience" kind. The real kind, where you find a market inefficiency, automate it, compound the returns and retire to a beach while the bots do the work.

Prediction markets looked perfect. Polymarket and Kalshi let people bet on real-world events like sports, crypto and politics, so surely someone was mispricing something.

So I did what any reasonable person would do: I built five daemons to monitor 1,600+ markets over three days, tested four statistical hypotheses with significance thresholds locked in advance, and paper-traded every signal that looked promising. Normal Tuesday stuff.

---

## The Question Underneath the Dream

Without the beach fantasy, the useful question is: **can I tell a real, tradeable edge apart from noise before I've put a dollar behind it?**

Most "edges" people report, in prediction markets or anywhere else, are overfitting in a hoodie. The only way to know whether mine was different was to lock the rules before looking at any results, and be willing to walk away from a number that looked good but didn't hold up.

---

## The Setup

I tracked four things:

- **Short-horizon**: are favorites/underdogs mispriced in <48h markets?
- **Flow tracker**: can I follow whale money and ride the move?
- **New market tracker**: is there a "stale window" where new markets are mispriced?
- **Player prop tracker**: are NBA player props systematically wrong?

Each hypothesis had the same bar, set before I collected a single data point:

- \>10% edge
- Statistically significant (p < 0.05)
- After transaction costs

No peeking and no adjusting mid-experiment. Honestly, this decision mattered more than anything that came after it.

---

## Hypothesis 1: Underdog Calibration

**The idea:** markets price underdogs at 30%, but they actually win 40%.

**What happened:**

- Day 0.5: +18% edge (n=31), which got me very excited
- Day 1: +12% edge (n=91)
- Day 2: +11% edge (n=165)
- Day 3: +7% edge (n=200+)

Textbook regression to the mean. By the time I had enough data to trust the signal, it had shrunk to +7%, which disappears completely after Polymarket's ~5% transaction costs.

**Verdict: dead.**

---

## Hypothesis 2: Flow Following

**The idea:** skip predicting outcomes, detect when smart money moves, and follow it.

I built a tracker for volume spikes, price velocity and whale-sized entries, and ran it across 300+ markets.

**Result:** a stable, consistent +6.5% edge, which is almost exactly what transaction costs eat. The edge exists, you just don't get to keep any of it.

**Verdict: dead.**

---

## Hypothesis 3: The Stale Window

**The idea:** new markets launch with naive prices and it takes hours for smart money to correct them, so bet early and capture the correction.

**What I found:**

- Sports markets show ~43% initial pricing error
- Most of the correction happens in the 6–24h window

That part was real and measurable. The problem is that the direction was random. Sometimes the initial price was too high and sometimes too low, with no consistent bias, so there was nothing to bet on, just a lot of noise.

**Verdict: dead.**

---

## Hypothesis 4: Player Prop OVERs

**The idea:** NBA player props are set by amateurs, so OVERs are systematically underpriced.

**What I found:**

- **Points O/U**: +7.5% bias, 44% win rate
- **Assists O/U**: +13.6% bias, 49% win rate
- **Rebounds O/U**: +11.7% bias, 47% win rate

This one held up. Assists and rebounds showed a consistent 11–14% mispricing, statistically significant across 200+ markets, with the correction happening in the 3–6h window. I'd found an edge, and I still wasn't celebrating, for reasons the paper trades made obvious.

---

## The Twist

Here's what paper trading showed:

- **Trades opened:** 8
- **Avg liquidity:** $17
- **Avg spread:** 84%
- **Trades with >$500 liquidity:** 0
- **Trades with <15% spread:** 1

An 84% spread means that if the "price" is 50%, you're really buying at 96% and selling at 4%. The +13% correction happens entirely inside a spread you can't cross. A typical market:

> Trendon Watford Assists O/U: $1 liquidity, 96% spread

I could see the inefficiency but had no way to trade it.

**Verdict: real edge, zero dollars.**

---

## The Meta-Lesson

This matters more than any of the four individual results:

> **Edges persist precisely where they cannot be traded.**

If the +13% player-prop edge were executable, arbitrageurs would have closed it years ago. The fact that it's still there is pretty good evidence that nobody can trade it.

Polymarket player props are a museum of inefficiencies. You can walk around and look at the exhibits, but you can't take anything home. The market is efficient where it needs to be, at the point of execution, and the mispricings everywhere else don't matter.

---

## The Scoreboard

- **Underdog calibration**: +7% edge → No (costs) → $0
- **Flow following**: +6.5% edge → No (costs) → $0
- **Stale window**: 43% error → No (random) → $0
- **Player prop OVERs**: +13% edge → No (liquidity) → $0

That's 3 days, 1,600 markets, 5 daemons and 4 hypotheses, for a grand total of 0 tradeable edges.

---

## Was It Worth It?

Yes, though not because I found alpha. I didn't, and now I know that from data rather than theory. I also know which failure killed each hypothesis (decay, cost, noise or liquidity), which is more useful than any single number would have been.

Most people lose money finding out that markets are efficient. I spent three days and $0 reaching the same conclusion, and got a blog post out of it, so I'd call that a profit.

---

## The Takeaway

If you're thinking about trying to beat prediction markets:

- You probably can't
- The real edges are either too small (costs) or too illiquid (spreads)
- That's the market working as intended, not a bug

If you find a "free money" opportunity, ask why it still exists. Usually it's because it isn't actually free.

The same three questions kill a fake edge here and a fake trading signal anywhere else: did it survive out-of-sample or just decay, does it clear costs, and can you actually execute it? The discipline carries over to any market.

I wanted passive income from market inefficiencies and got a lesson in market efficiency instead. Now if you'll excuse me, I need to go find another way to not make money.
