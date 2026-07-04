---
layout: post
title: "Confidently Wrong in Production: A Spark-on-EKS Postmortem on Agent Trust Boundaries"
date: 2026-07-04
description: "A real onboarding incident on Spark/EKS, and the verification boundary that caught what an AI coding assistant confidently got wrong."
---

We were about to onboard a new team in production to run Spark jobs on EKS. Routine, on paper — we already had hundreds of teams running on the platform.

They had a couple of custom libraries. I baked them into the base EMR image before their first job launch, standard procedure. We tested the exact same setup in dev heavily — load tests, performance tests, the works. Everything passed.

Before hopping on a call to walk the team through the prod CI/CD flow, I ran a dummy job in production using the baked image. Same image as staging. Byte-for-byte.

That's when it got interesting.

---

## Everything Breaks (Immediately)

The job doesn't even reach the Kubernetes controller.

Pods just sit there. **Pending.**

My Claude Code session had hit its limit, so I was working the incident in Cursor. I asked it what was going on.

Cursor confidently told me:

- Executor registration is failing
- I'm using the wrong image
- The entrypoint is wrong
- `spark-submit` isn't working for executors

Every engineer knows the feeling that follows: *but it literally worked yesterday.*

I asked Cursor to double-check pod status. Same answer. Categorical. Absolute.

> **Wrong image. Period.**

---

## Reality Check (a.k.a. Logs Don't Lie)

So I checked the logs myself — I keep a few MCP servers around specifically to make log-fetching fast, and this is exactly the situation they're for. I pulled Kubernetes logs directly.

**Capacity issue.** The memory allocation for the job was too high, and Karpenter wasn't provisioning nodes fast enough. Not Karpenter's fault — a different postmortem.

I told Cursor to reduce RAM to 700MB and run a single executor. It pushed back: "that's not the problem." I overrode it and ran the job anyway.

It failed again. I checked Cursor. Still categorical:

> Executors didn't register. Wrong image. I told you.

Meanwhile my MCP server was pointing at something else entirely: a **schema mismatch** in the code. Cursor missed it, again. I fixed the schema, re-ran the job.

Success. Cursor's response: no acknowledgment, no correction, just "the job succeeded."

---

## Where the Real Risk Lives

If a less experienced engineer had been in that seat, trusted the first confident answer, and burned hours chasing a nonexistent image problem while the actual issues — capacity, then schema — went unaddressed, that's not a hypothetical. That's a real production incident, the kind that costs a team its first-week confidence in a new platform.

The danger isn't that an agent gets something wrong. It's that when an LLM hits the edge of what it knows, it doesn't say "I don't know" — it fills the gap with something plausible-sounding and states it with the same confidence as something it actually verified. There's no signal in its tone that distinguishes a guess from a fact.

---

## The Boundary That Actually Matters

The lesson isn't "don't use agents in production infra." It's that the useful question isn't whether an agent is right — it's **what it's allowed to touch before a human confirms.**

In this incident, the boundary that saved the debugging session wasn't a smarter model — it was a separate, independent path to ground truth (the MCP-based log access) that didn't depend on the agent's own diagnosis being correct. The agent proposed; the logs disposed.

The operating rule I take from this: an agent can read, observe, and propose anything. It doesn't get to be the only source of truth for a diagnosis, and it doesn't get to take an irreversible action — restart, rollback, resource change — without something outside the agent's own reasoning confirming it first. Not because the agent is untrustworthy in general, but because confidence and correctness are two different signals, and only one of them is visible in the response.

You're still the owner of the system. The agent is an increasingly useful pair of hands. It is not, yet, a second source of truth.
