---
layout: post
title: "When Cursor Was Confidently Wrong in Production"
date: 2026-02-03
description: "A real debugging story where AI guidance pointed everywhere except the actual problem."
---

I'm back for a new episode.

This time, I want to talk about an interesting (and mildly infuriating) experience I had with Cursor.

To be clear: I *like* Cursor. Or… I liked it—in a very specific way.

I like that it's built on top of a familiar tool like VS Code. It's interesting. It's genuinely better than Copilot will ever be (seriously—what *is* going on at Microsoft these days?). Overall, I'm cool with Cursor.

I use it occasionally for **low-cognitive-load tasks**:

- Formatting CloudWatch logs
- Checking Kubernetes pod statuses
- Inspecting Kafka connectors

Nothing crazy. (Yes, I have a few MCP servers lying around to make log-fetching easier. Occupational hazard.)

---

## The Setup (a.k.a. Nothing Exotic)

On this particular day, my Claude Code session hit its limit, so I switched to Cursor.

We were about to onboard a new team in production to run Spark jobs on EKS. No big deal. This is routine.

They had a couple of custom libraries. I made sure they were baked into the base EMR image *before* their first job launch. Standard procedure.

We tested the exact same setup in dev—*heavily*. Load tests, performance tests, the whole thing. Everything passed.

At this point, we already had **hundreds of satisfied customers** running on the platform. So yeah, confidence was high.

Before hopping on a call with the team to walk them through the prod CI/CD flow, I decided to run a dummy job in production using the baked image.

Same image as staging. Byte-for-byte.

And then…

---

## Everything Breaks (Immediately)

The job doesn't even reach the Kubernetes controller.

Pods just sit there.

**Pending.**

Great.

So I ask Cursor:

> "Hey man, what's up?"

Cursor confidently tells me:

- Executor registration is failing
- I'm using the wrong image
- The entrypoint is wrong
- `spark-submit` isn't working for executors

Now look—this *can* happen. But every engineer knows this feeling:

> *"But… it literally worked yesterday."*

I ask Cursor to double-check pod status.

Same answer.

Categorical. Absolute. No doubt whatsoever.

> **Wrong image. Period.**

At this point, I'm perplexed.

---

## Reality Check (a.k.a. Logs Don't Lie)

So I do what engineers still have to do in 2026: I check the logs myself.

I spin up my MCP server and pull Kubernetes logs directly.

And guess what?

**Capacity issue.**

The memory allocation for the job was too high. Karpenter wasn't provisioning nodes fast enough.

(Not Karpenter's fault, by the way—but that's a different blog post.)

So I tell Cursor:

> "Reduce RAM to 700MB and run a single executor."

Should work.

Cursor pushes back.

> "That's not the problem."

I override it anyway. (Which means I'm now fighting **Cursor + EKS + Spark** simultaneously. Not my best day.)

The job runs.

I monitor logs in parallel via MCP.

The job fails.

I check Cursor.

Once again, it's *categorical*:

> Executors didn't register. Wrong image. I told you.

I swear it smirked.

---

## The Missed Signal

Meanwhile, my MCP server is screaming something else entirely:

**Schema mismatch.**

Fix the code.

Cursor missed it. Again.

I fix the schema.

Re-run the job.

**Success.**

Cursor's response?

No acknowledgment. No correction. No "hey, I was wrong."

Just:

> "The job succeeded."

Cool.

---

## The Real Risk No One Talks About

If a non-experienced engineer had been in that seat, trusted Cursor, and spent hours debugging a **nonexistent image problem** while the real issues—capacity constraints first, then a schema mismatch—went unnoticed?

That's not theoretical.

That's a **real production incident waiting to happen**.

Confidently wrong guidance is not just annoying—it's dangerous when it diverts attention away from the actual failure mode.

---

## So… What Does This Tell Us?

Should we ditch agentic AI in production?

**No. Absolutely not.**

Should we let it run the show?

**Also no. Absolutely not.**

Not in production.
Not in staging.
Not anywhere that matters.

Here's the real takeaway:

> **You, as the engineer, must know what you're doing.**

When an LLM gets stuck, it doesn't say *"I don't know."* It fills the gap.

Fast conclusions.
Wrong conclusions.
Confident conclusions.

If you relinquish control, it *will* dupe you—not maliciously, but mechanically.

AI is an assistant.

You are the owner.

And ownership is still very much a human responsibility.
