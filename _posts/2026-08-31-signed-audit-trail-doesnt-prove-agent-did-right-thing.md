---
layout: post
title: "A Signed Audit Trail Doesn't Prove Your AI Agent Did the Right Thing"
date: 2026-08-31
description: "AgentGate is a useful example of why recording an agent's actions and governing those actions are two different problems."
---

"Signed audit logs" sounds like a solved problem. Hash the record, sign it, and nobody can dispute what happened. For AI agents that misses the point. The important question isn't whether the log is tamper-evident after it's written, it's whether the agent can influence, skip, or route around whatever produces the log. An audit trail and a governance control are different things, and once cryptography is involved it's easy to mix them up.

A small open-source project called [AgentGate](https://github.com/Clawdlinux/agentgate), a gateway that generates signed receipts for actions an AI agent takes against SaaS APIs like GitHub, Slack and Google Workspace, turned out to be a good way to make that distinction concrete. I'm going to take its documentation at face value, go through what it actually establishes, and then ask the question that matters if you're thinking about giving an agent real authority: is receipt generation outside the agent's control, or is it a logging call the agent has simply agreed to make?

I've hit a version of this before in a different domain. I [built a small multi-agent system for catching drift in a market-data pipeline](https://youcefjd.github.io/youcefdjeddar/teaching-agents-to-watch-a-pipeline/) around one rule: agents can see everything, but what a compliance reviewer trusts is the audit trail, not the agent's own account of what it did. An agent describing its own behavior isn't evidence. Anything a skeptical reviewer relies on has to be verifiable independently of the system being reviewed. AgentGate makes the same bet for SaaS API calls instead of pipeline monitoring, which is why I think it's worth looking at properly rather than writing it off as another audit-log wrapper with a hash on top.

Here's what AgentGate's documentation establishes that matters for this argument. The agent never holds OAuth credentials. The gateway holds all tokens, and the agent's only way to reach any upstream API is a single endpoint. Every action attempt, including failures, writes a signed, hash-chained receipt before the response goes back to the agent. And anyone can verify that receipt chain offline, without access to the gateway's internals, using only a copy of the log and a pinned trust file.

That last part is the important one. Because the receipt is written as part of handling the request, not as a separate call the agent makes afterward, an agent inside this architecture has no way to reach the upstream API without producing a record. That's a real structural property of the design, but it isn't a guarantee about any particular deployment, and the documentation doesn't claim it is. It doesn't protect against a deployment that also gives the agent direct credentials by mistake, or one that doesn't force all outbound traffic through the gateway at the network level. "The agent can't bypass the receipt" is true of the design, and only true in production if the surrounding infrastructure enforces it.

When people say "audit trail" they usually lump four separate properties together:

**Non-repudiation**: proof of who or what produced a record. A signature ties a receipt to whoever holds the signing key.

**Tamper-evidence**: proof a record wasn't changed afterward. A hash chain makes retroactive edits detectable.

**Completeness**: proof nothing was skipped. Writing receipts for failed attempts too, plus a way to checkpoint that a section of the chain has no gaps, is aimed at this.

**Correctness of the decision**: whether the agent should have taken the action at all. None of the above addresses this. A perfectly signed, gap-free, tamper-evident log can accurately record an agent doing something disastrous. It tells you what happened and that the record wasn't altered, and nothing about whether the action was authorized, reasonable, or within policy. To its credit, AgentGate's documentation says this is out of scope.

I started out thinking most "agent audit trail" tools quietly assume the agent will cooperate with its own monitoring. That's true of some tools here but not all of them. A voluntary logging SDK is easy to defeat: an agent that doesn't call it, or calls it with made-up parameters, leaves no trace, because the SDK has no control over whether the action happens. AgentGate works differently. The gateway is the only path to the credentials, so the receipt isn't optional telemetry about something that happened elsewhere. It's produced by the same component that makes the call. It wouldn't be fair to group it with tools that are just self-reported logging with a hash attached.

Where my original criticism does still apply is the gap between design and deployment. The guarantee depends on the agent having no other path to the same upstream APIs, and that's a question of network topology and operational discipline, not of the gateway's code. In production, the deploying organization has to block every route from the agent's runtime to those APIs except through the gateway, and no gateway can enforce that from inside its own process. So the fair version of my critique is narrower than "this tool assumes cooperation." It's that the guarantee is only as strong as the deployment's network policy.

This matters more as agents start taking actions that are hard to reverse or have real financial consequences: updating a record that feeds a downstream decision, starting a transfer, calling a tool that silences an alert. Even taking the design entirely at its word, there's one gap I'd want answered before using something like this somewhere high-stakes. The documentation doesn't say how the trust root gets rotated if it's compromised, or who owns that process. For a system whose whole value depends on verifying offline against a pinned trust root, that's a real gap.

After reading the actual documentation instead of just the pitch, my argument ended up narrower than where I started, and I think more useful. A signed receipt only works as a governance control if generating it is structurally outside the agent's reach. On that point this design deserves more credit than most "audit trail for agents" pitches, since as documented it isn't a call the agent can skip. What it doesn't do, and doesn't claim to do, is tell you whether the action was correct, who's accountable if the signing key leaks, or whether the deployment actually closes off the side channels its guarantees depend on. Those are network-policy and key-management questions as much as cryptographic ones.

The broader point goes beyond this one project. "We log everything and sign it" keeps getting treated as an answer to the governance question, when it only answers the evidence question. Those are different problems, and mixing them up is how a well-built audit trail ends up being treated as oversight it was never designed to provide.
