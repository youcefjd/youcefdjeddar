---
layout: post
title: "Confidently Wrong in Production: A Spark-on-EKS Postmortem on Agent Trust Boundaries"
date: 2026-07-04
description: "A real onboarding incident on Spark/EKS, and the verification boundary that caught what an AI coding assistant confidently got wrong."
---

I'm back for a new episode. This time I want to talk about an interesting (and mildly infuriating) experience I had with Cursor.

To be clear, I *like* Cursor, or at least I liked it in a very specific way. It's built on top of VS Code, which I already know, and it's better than Copilot will ever be (seriously, what *is* going on at Microsoft these days?). Overall I'm fine with it.

I use it occasionally for **low-cognitive-load tasks**:

- Formatting CloudWatch logs
- Checking Kubernetes pod statuses
- Inspecting Kafka connectors

Nothing crazy. (Yes, I have a few MCP servers lying around to make log-fetching easier. Occupational hazard.)

---

## The Setup (a.k.a. Nothing Exotic)

On this particular day my Claude Code session hit its limit, so I switched to Cursor.

We were about to onboard a new team in production to run Spark jobs on EKS, which is routine for us. They had a couple of custom libraries, and I made sure those were baked into the base EMR image *before* their first job launch, as usual.

We'd tested the exact same setup heavily in dev: load tests, performance tests, the whole thing, and everything passed. We already had **hundreds of satisfied customers** on the platform, so confidence was high.

Before hopping on a call to walk the team through the prod CI/CD flow, I ran a dummy job in production with the baked image, byte-for-byte the same image as staging. It broke immediately.

---

## Everything Breaks (Immediately)

The job didn't even reach the Kubernetes controller. The pods just sat in **Pending**.

So I asked Cursor:

> "Hey man, what's up?"

Cursor told me, with total confidence, that:

- Executor registration was failing
- I was using the wrong image
- The entrypoint was wrong
- `spark-submit` wasn't working for executors

That kind of thing *can* happen, but every engineer knows the feeling of "it literally worked yesterday." I asked Cursor to double-check the pod status and got the same answer, with no doubt whatsoever:

> **Wrong image. Period.**

At this point I was confused.

---

## Reality Check (a.k.a. Logs Don't Lie)

So I did what engineers still have to do in 2026 and checked the logs myself. I used my MCP server to pull the Kubernetes logs directly, and it was a **capacity issue**. The job's memory allocation was too high and Karpenter wasn't provisioning nodes fast enough. (Not Karpenter's fault, by the way, but that's a different blog post.)

So I told Cursor:

> "Reduce RAM to 700MB and run a single executor."

Cursor pushed back:

> "That's not the problem."

I overrode it anyway, which meant I was now fighting **Cursor, EKS and Spark** at the same time. Not my best day.

The job ran, and I watched the logs in parallel through MCP. The job failed, and Cursor was once again completely sure of itself:

> Executors didn't register. Wrong image. I told you.

I swear it smirked.

---

## The Missed Signal

Meanwhile the MCP logs were pointing at something else entirely: a **schema mismatch** that needed a code fix. Cursor missed that too.

I fixed the schema, re-ran the job, and it succeeded. Cursor didn't acknowledge it had been wrong at any point. All it said was:

> "The job succeeded."

Cool.

---

## The Real Risk No One Talks About

Imagine a less experienced engineer in that seat who trusted Cursor and spent hours chasing a **nonexistent image problem** while the real issues, first capacity and then a schema mismatch, went unnoticed. That's a very plausible **production incident**.

Wrong guidance delivered with confidence is worse than annoying, because it pulls your attention away from what's actually failing.

---

## So… What Does This Tell Us?

Should we ditch agentic AI in production? No. Should we let it run the show? Also no, not in production, not in staging, and not anywhere else that matters.

The takeaway for me is that **you, the engineer, still have to know what you're doing.**

When an LLM gets stuck, it doesn't say "I don't know." It fills the gap with a quick conclusion, and nothing in its tone tells you whether that conclusion is a guess or a verified fact. The mistake itself isn't really the dangerous part. The dangerous part is that a guess and a fact sound exactly the same. If you hand over control, it will mislead you eventually, not out of malice but just because of how it works.

What saved this debugging session wasn't a smarter model. It was having a separate, independent way to see what was really happening (the MCP-based log access) that didn't depend on Cursor's diagnosis being right. Cursor suggested causes, and the logs settled which ones were real.

The rule I took from this: an agent can read, observe and propose anything it likes. It doesn't get to be the only source of truth for a diagnosis, and it doesn't get to take an irreversible action (a restart, a rollback, a resource change) unless something outside its own reasoning confirms it first.

AI is a useful assistant, but owning production is still a human job.
