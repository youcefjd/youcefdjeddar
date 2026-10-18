---
layout: post
title: "Teaching Agents to Watch a Data Pipeline Without Trusting Them Blindly"
date: 2026-07-05
description: "A small multi-agent system for catching market-data pipeline drift, built around a rule: agents can see everything, but the audit trail is what a compliance reviewer actually trusts."
---

In the last post, what saved a bad debugging session was a boundary, not a smarter model. Cursor could propose whatever it wanted, but it couldn't act on any of it without something outside its own reasoning confirming it first.

That got me wondering about the reverse setup. Instead of an agent proposing fixes that a human checks, what if the agent's whole job is watching, and the thing being watched is a data pipeline that can't afford to fail quietly?

So I built a small one, using a synthetic market-data feed over the same Kafka pattern I've used before, to see whether the boundary idea still holds when the agent is a monitor instead of a coding assistant.

---

## The Problem With Watching Pipelines

Market-data pipelines fail in boring, specific ways. A schema changes upstream and nobody tells you. A producer starts dropping fields silently. Consumer lag creeps up until a downstream job is working off stale data. A vendor feed goes quiet without any error and just stops.

The usual fix is threshold alerting, which tends to be wrong in one of two directions. Make it too sensitive and you get paged for noise until you tune it down. Make it too quiet and that tuning is exactly what lets the real incident through. The alerting logic isn't the problem in either case. One metric crossing one threshold is just a poor stand-in for "something is actually wrong."

What you really want is something that looks at several weak signals together, decides whether they add up to something worth a human's attention, and leaves an exact record of why it decided that. That last part is what anyone who has to trust it will care about.

It's worth remembering how bad things get when nothing is watching at all. Knight Capital lost $440 million in 45 minutes on August 1, 2012. A deployment rolled new code to seven of eight production servers and left an old, dormant order-routing function alive on the eighth, which got reactivated by a repurposed flag that nobody had traced all the way through. Nothing crashed and nothing errored. It just sent the wrong orders, very fast, and nobody caught it until the damage was done. That's far bigger than a schema-drift alert, but the root cause is similar: a silent divergence that nobody was independently watching for, in a system where the quiet failures are the expensive ones.

---

## The Shape of It

There are three agents, each with a narrow job and an even narrower set of tools:

- **Detectors**: one per signal type (schema-registry version history, consumer lag, message-rate volume). Each has read access to its own metric only. No detector can see another detector's data, and none can take any action.
- **Triage**: reads the detectors' output, correlates it, and decides whether this is noise, a low-severity anomaly, or something a human needs to see now. It has no tools at all and only reasons over what the detectors reported.
- **Narrator**: takes triage's decision and writes the plain-English incident summary a human would actually read. It can't change the severity or reopen the analysis. Its only job is making the record readable.

None of the three can restart a consumer, roll back a schema, or touch the pipeline in any way, and that's on purpose. The system's job is to produce a claim you can trust ("here's what's wrong and why we think so"), not to act on it. Acting is still a human decision, same as in the EKS post.

---

## What "Scoped" Actually Looks Like

The boundary isn't enforced by a prompt asking the agent to behave. The tool simply doesn't exist. A detector's MCP tool definition looks roughly like this:

```python
@mcp_tool(name="read_consumer_lag")
def read_consumer_lag(topic: str, window_minutes: int = 15) -> LagSnapshot:
    """
    Read-only. Returns lag metrics for a single topic over the given window.
    No write, restart, or config-change capability is exposed on this server.
    """
    return kafka_admin_client.get_consumer_lag(topic, window_minutes)

# Deliberately absent from this server: restart_consumer, patch_topic_config,
# or anything else that mutates pipeline state. If it isn't a defined tool,
# the agent cannot call it — there's no fallback path to "just try it anyway."
```

If an incident really does call for restarting a consumer or rolling back a schema, that happens in a separate, human-operated system. The monitoring agents never get a path to it.

---

## The Part That Makes It Auditable

Every step gets written to an append-only log before the narrator sees anything: which detector fired, what evidence it returned, what triage decided, and with what confidence. The log holds the decision together with the inputs it was based on, not just a summary.

```json
{
  "run_id": "run-2026-07-11T02:14:00Z",
  "agent": "triage",
  "inputs": [
    {"detector": "schema_registry", "signal": "unversioned field drop: settlement_ccy", "confidence": 0.82},
    {"detector": "consumer_lag", "signal": "lag 40s -> 190s over 15m window", "confidence": 0.61}
  ],
  "decision": "escalate",
  "severity": "medium",
  "rationale": "Schema drift on a settlement-relevant field correlated with rising lag on the same topic within the same window — treating as one incident, not two.",
  "human_action_required": true
}
```

A compliance or risk reviewer replaying this doesn't have to take the system's judgment on faith. They can see which two signals triage was looking at and why it linked them, then reach the same conclusion from the same evidence, or disagree with it. Being able to disagree with it is the whole reason for keeping the log.

---

## What This Isn't

This is a prototype built against a synthetic feed over a few days, not a production system, and there's a fair amount missing before it could be one. I haven't measured its false-positive rate over a long period, it hasn't been tested against adversarial or truly ambiguous drift patterns, and it isn't connected to a real incident-management tool. The narrator's output currently goes to a log file, not a pager.

What I did check is the thing I set out to test. The pattern of "agents can see everything, but a narrow, explicit set of tools limits what they can touch" works outside of coding assistants too. And logging the evidence behind a decision, not just the decision, is what turns "the agent flagged this" into something a skeptical reviewer can verify.

---

## The Principle, Restated

It's the same rule as last time, applied one level up. For an agent, the design questions that matter most are what it's allowed to touch, and whether someone else can independently check the reasoning behind what it flagged. How clever its reasoning is matters a lot less.
