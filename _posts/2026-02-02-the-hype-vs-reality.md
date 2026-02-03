---
layout: post
title: "The Hype vs Reality of AI Replacing Engineers"
date: 2026-02-02
description: "Why tech CEOs need to sell inevitability, and why you shouldn't believe them."
---

**First, let me be clear:** I'm pro-AI.

I've been working on ML since 2018—right when transformers went mainstream, long before the LLM tsunami. I've done computer vision, NLP, RNNs (yes, those), and plenty of unglamorous production work. If anything, I'm deeply invested in AI.

What I *don't* subscribe to is the hyperventilating hype around GenAI—especially the casual claims about what it means for software engineers and the broader engineering ecosystem.

Every day you hear tech CEOs on X (because who says anything important on LinkedIn anyway?) declaring that *you'll lose your job in a year*. Cursor writes faster code than you. One engineer + AI will replace three senior devs. Why pay someone to manage Kafka when your boss can just prompt an LLM?

Of course that sounds threatening. When Amodei, Altman, Huang, etc. say this loudly and repeatedly, your boss is listening. Your boss's boss is listening. And they're thinking: *why am I paying all these people?*

If I were your boss's boss, I'd tell them to shut that intrusive thought down immediately.

Why?

Because **Amodei *needs* to say this.**

He *has* to sell inevitability. You don't raise $100B checks by saying, "Progress is incremental, expensive, and constrained by reality." You raise it by promising a revolution. That's the incentive structure. Full stop.

I'll write more later about my firsthand experience running AI in high-velocity production environments, but for now, let's talk reality.

---

## The Plateau Is Real

Despite the hype, evidence from inside the industry shows diminishing returns in raw LLM capability.

- **Benchmarks are flattening.** GPT-4o shows ~3–7% gains over GPT-4 on metrics like MMLU and GSM8K—compared to 15–25% jumps in earlier generations.
- **High-profile models missed expectations.** OpenAI's Orion and Google's Gemini 2.0 underdelivered relative to internal targets.
- **Even the architects admit it.** Ilya Sutskever said at NeurIPS 2024 that *LLM scaling has plateaued*, particularly for non-reasoning tasks.
- **Open source is catching up.** Despite absurd capital expenditure, proprietary models aren't pulling away the way they used to.

Yes, models are improving—but at **enormous cost** for **marginal gains**.

---

## Where Progress Actually Moved: Agentic AI

As scaling stalled, progress shifted sideways.

Agentic systems—chaining LLMs together into workflows—*do* unlock new capabilities:

- Multi-step task execution
- Tool use and orchestration
- Systems like CrewAI, LangGraph, Claude Code, etc.

But notice something important:

> This doesn't eliminate engineering. It **adds more of it**.

Agentic AI scales *depth*, not intelligence. It requires architecture, constraints, testing, observability, and constant human oversight. The breakthroughs are in **efficiency, latency, cost**, and **multimodality**—not raw "text smarts."

---

## Okay, So… Now What?

Here's the part people miss.

AI has genuinely lowered the barrier to building things. Non-technical folks can now create apps, prototypes, and workflows that would've been impossible for them five years ago. That's a real win.

But let's be honest.

LLMs are not replacing the engineers who actually **scale systems**, **own production**, and **carry operational responsibility**.

I don't write much syntax anymore. I let AI handle boilerplate and lint-level thinking. But I write the tests. I design the systems. I decide what *must not* happen. I use agentic AI—especially Claude Code—as an **assistant**.

Giving it the reins would be a fatal mistake. (I'll explain why in a future post, based on a very real production incident.)

Amodei can say whatever he wants. He has a product to sell.

You don't.

Your job is to **ally with AI**, not fight it. Use it to become faster, sharper, and more dangerous—not to outsource judgment, responsibility, or ownership.

The engineers who disappear won't be replaced by AI.

They'll be replaced by engineers who know how to *use* it—without believing the hype.
