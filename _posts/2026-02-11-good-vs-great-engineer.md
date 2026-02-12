---
layout: post
title: "What Separates a Good Engineer from a Great Engineer"
date: 2026-02-11
description: "Hint: it's not distributed systems wizardry."
---

Every few months someone asks: what separates a good engineer from a great one?

Most answers point to technical brilliance — system design mastery, elegant abstractions, distributed systems wizardry.

Cool story.

Not the real answer.

---

## It's Not About Complexity

Great engineers don't build the most complex systems.

Anyone can assemble an impressive stack of queues, stream processors, and microservices. Complexity is easy — especially with modern tooling. (Even easier now that you can just ask Claude to scaffold it for you.)

Restraint is hard.

Elegance isn't about cleverness. It's about solving the problem *exactly* — no more, no less.

Not overbuilt.
Not underbuilt.
Not designed for a hypothetical future that never arrives.

Just right.

You cannot build the right system if you don't fully understand the problem.

Which means engineering excellence has less to do with code than most people think.

Controversial? Maybe. True? Absolutely.

---

## Impact Is the Only Scoreboard

You can build the most technically impressive system in the company.

Perfect latency.
Beautiful architecture.
Flawless reliability.

If nobody uses it — or nobody needs it — you failed.

Sorry.

Engineering isn't judged by technical beauty.

It's judged by impact.

Impact comes from alignment with reality:

- What the product actually needs
- What the business actually values
- What stakeholders are trying to accomplish

Not what sounds impressive in a design doc. Not what gets you compliments in code review. Not what looks good on your LinkedIn.

---

## Speak the Stakeholder's Language

Great engineers understand the product as deeply as they understand the tech.

They don't hide behind jargon.
They don't stay confined to engineering meetings.
They don't wait for perfectly written requirements. (Spoiler: they never come.)

They go where context lives.

Product reviews.
Planning meetings.
Strategy calls.
Internal stakeholder syncs.

Even when attendance is optional.

They listen.
They ask questions.
They learn how the business actually works.

Every hour spent understanding the problem saves days building the wrong solution.

I know, I know — meetings bad. But you know what's worse? Building something nobody asked for.

---

## Your Job Is Also to Refine the Problem

Stakeholders rarely arrive with perfectly formed requirements.

Not because they're careless — because translating business intent into technical constraints is hard. That's literally why they hired you.

Great engineers don't just execute requests.

They refine them.

They ask:

- Why this requirement?
- What decision depends on this?
- What breaks if latency is 5 seconds instead of 500ms?
- Who actually uses this output?

Sometimes the honest answer is: nothing breaks.

That changes everything.

---

## Fancy Is Not the Same as Necessary

A stakeholder once asked for sub-second latency for a system analyzing customer behavior for advertising.

Sub-second sounded serious. Advanced. Impressive.

There was just one issue: no one had a real use case that required it.

Achieving true sub-second processing would have meant:

- Replacing Spark with Flink
- Re-architecting the transformation pipeline
- Scaling Kafka ~3×
- Increasing operational complexity across the board

We're talking terabytes of data and permanent infrastructure cost. The kind of project that spawns three new Jira epics and a dedicated Slack channel.

When we mapped the actual decision loop — how often campaigns changed, how quickly humans reacted, how the data was consumed — it became clear:

Spark Structured Streaming with micro-batching was enough.

Not theoretically optimal.
Not ultra-low latency.
But perfectly aligned with reality.

Fancy would have been expensive theater.

Appropriate was correct.

The stakeholder didn't need sub-second. They needed *someone to tell them they didn't need sub-second*.

---

## The Senior → Staff Shift

This is where the real transition happens.

Senior engineers build solid systems.

Staff engineers align systems with reality.

They:

- Translate business intent into technical strategy
- Push back on unclear or excessive requirements
- Prevent over-engineering before it starts
- Guide stakeholders toward sane solutions
- Create clarity where none exists

Most importantly, they bring people along.

You can design the most rational architecture in the world. If stakeholders don't trust or understand it, it won't matter. You'll spend six months in committee and ship nothing.

Great engineers build systems.

Exceptional engineers build alignment.

---

## The Skill Nobody Teaches

We train engineers to:

- Optimize systems
- Scale infrastructure
- Write better code
- Design better abstractions

We rarely train them to:

- Understand business context
- Challenge requirements
- Communicate trade-offs
- Influence decisions
- Earn trust

Yet those skills determine whether your work has any real impact.

Leetcode won't teach you this. Neither will your CS degree. You learn it by paying attention to the parts of the job that aren't coding.

---

## Why AI Won't Replace This

This is also why AI won't replace senior engineers anytime soon.

I've [written about this before](/the-hype-vs-reality-of-ai-replacing-engineers/) — the gap between what AI can do and what the hype claims is massive.

LLMs are great at "build me X." They're terrible at "should we build X at all?"

That question requires understanding the business, the stakeholders, the actual decision loops. It requires sitting in meetings, reading between the lines, and knowing when someone says "sub-second latency" but means "I want to feel like this is a serious project."

It's the least automatable part of the job.

And it's the part that actually matters.

---

## Bottom Line

What separates a good engineer from a great one isn't raw technical ability.

It's judgment.

Knowing what to build.
Knowing what not to build.
Knowing how to align technology with reality.
Knowing how to guide stakeholders to the right solution — even when they don't yet see it.

The best system isn't the most sophisticated.

It's the one that solves the right problem, at the right time, in the right way — and actually gets used.

Everything else is resume padding.
