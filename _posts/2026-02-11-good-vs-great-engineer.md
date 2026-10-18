---
layout: post
title: "What Separates a Good Engineer from a Great Engineer"
date: 2026-02-11
description: "Hint: it's not distributed systems wizardry."
---

Every few months someone asks what separates a good engineer from a great one. Most answers point to technical brilliance: system design mastery, elegant abstractions, distributed systems wizardry. I don't think that's it.

---

## It's Not About Complexity

Great engineers don't build the most complex systems. Anyone can assemble an impressive stack of queues, stream processors and microservices, especially with modern tooling (and even more so now that you can ask Claude to scaffold it for you).

The hard part is restraint. A good solution solves the problem you actually have, without being overbuilt, underbuilt, or designed for some hypothetical future that never shows up. You can't get there if you don't fully understand the problem, which means engineering excellence has less to do with code than most people think.

---

## Impact Is the Only Scoreboard

You can build the most technically impressive system in the company, with perfect latency, beautiful architecture and flawless reliability, and if nobody uses it or nobody needs it, it was a waste.

Engineering gets judged by impact, and impact comes from lining up with what's real:

- What the product actually needs
- What the business actually values
- What stakeholders are trying to accomplish

Whether it sounds impressive in a design doc, gets compliments in code review, or looks good on LinkedIn doesn't count for much.

---

## Speak the Stakeholder's Language

Great engineers understand the product as well as they understand the tech. They don't hide behind jargon, they don't stay inside engineering meetings, and they don't wait for perfectly written requirements, because those never arrive.

They go where the context is: product reviews, planning meetings, strategy calls, stakeholder syncs, even the ones where attendance is optional. They listen, ask questions, and learn how the business actually works. An hour spent understanding the problem can save days of building the wrong thing.

I know nobody likes meetings. Building something nobody asked for is worse.

---

## Your Job Is Also to Refine the Problem

Stakeholders rarely show up with perfectly formed requirements. That's usually not carelessness. Translating business intent into technical constraints is hard, and it's a big part of why they hired engineers in the first place.

So instead of just executing the request, a great engineer asks questions like:

- Why this requirement?
- What decision depends on this?
- What breaks if latency is 5 seconds instead of 500ms?
- Who actually uses this output?

Sometimes the honest answer is that nothing breaks, and that changes the whole project.

---

## Fancy Is Not the Same as Necessary

A stakeholder once asked for sub-second latency on a system analyzing customer behavior for advertising. It sounded serious, but nobody had a real use case that needed it.

Getting to true sub-second processing would have meant:

- Replacing Spark with Flink
- Re-architecting the transformation pipeline
- Scaling Kafka ~3×
- Adding operational complexity everywhere

This was terabytes of data and permanent infrastructure cost, the kind of project that spawns three Jira epics and its own Slack channel.

When we mapped the actual decision loop (how often campaigns changed, how quickly people reacted, how the data was consumed), it was clear that Spark Structured Streaming with micro-batching was enough. It wasn't the theoretically optimal or lowest-latency option, but it matched how the system was really used. The fancy version would have been expensive theater.

What the stakeholder really needed was someone to tell them they didn't need sub-second.

---

## The Senior → Staff Shift

This is where a lot of the senior-to-staff transition happens. Senior engineers build solid systems. Staff engineers make sure the systems match what the business needs. In practice that means they:

- Translate business intent into technical strategy
- Push back on unclear or excessive requirements
- Prevent over-engineering before it starts
- Guide stakeholders toward sane solutions
- Create clarity where there isn't any

Most importantly, they bring people along. You can design the most rational architecture in the world, but if stakeholders don't trust it or understand it, you'll spend six months in committee and ship nothing.

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

Those second skills decide whether your work has any real impact. Leetcode won't teach them and neither will a CS degree. You pick them up by paying attention to the parts of the job that aren't coding.

---

## Why AI Won't Replace This

This is also why I don't think AI replaces senior engineers anytime soon. I've [written about this before](/youcefdjeddar/the-hype-vs-reality/): the gap between what AI can do and what the hype claims is huge.

LLMs are great at "build me X" and bad at "should we build X at all?" Answering that takes understanding the business, the stakeholders and the actual decision loops. It means sitting in meetings, reading between the lines, and noticing when someone says "sub-second latency" but means "I want this to feel like a serious project." That's the least automatable part of the job, and in my experience it's the part that matters most.

---

## Bottom Line

What separates a good engineer from a great one is mostly judgment: knowing what to build, what not to build, and how to get stakeholders to the right solution even when they don't see it yet.

The best system usually isn't the most sophisticated one. It's the one that solves the right problem at the right time and actually gets used. A lot of the extra sophistication is resume padding.
