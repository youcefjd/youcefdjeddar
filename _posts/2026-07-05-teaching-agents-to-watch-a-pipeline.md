---
layout: post
title: "Teaching Agents to Watch a Data Pipeline Without Trusting Them Blindly"
date: 2026-07-05
description: "A small multi-agent system for catching market-data pipeline drift, built around a rule: agents can see everything, but the audit trail is what a compliance reviewer actually trusts."
---

In the last post, the thing that saved a bad debugging session wasn't a smarter model — it was a boundary. Cursor could propose whatever it wanted; it couldn't act on any of it without something outside its own reasoning confirming it first.

That got me wondering what happens if you flip the setup: instead of an agent proposing fixes that a human checks, what if the agent's entire job *is* watching — and the thing being watched is a data pipeline that can't afford to be wrong quietly?

So I built a small one, using a synthetic market-data feed over the same Kafka pattern I've used before, to see whether the "boundary, not smarter reasoning" idea actually holds up when the agent isn't a coding assistant anymore but a monitor.

---

## The Problem With Watching Pipelines

Market-data pipelines fail in boring, specific ways: a schema changes upstream and nobody tells you, a producer starts dropping fields silently, consumer lag creeps up until a downstream job is working off stale data, a vendor feed goes quiet without an error — it just stops.

The usual fix is threshold alerting. It's also usually wrong in one of two directions: too sensitive, and you get paged for noise until you tune it down; too quiet, and the tuning-down is exactly what lets the real incident through unnoticed. Neither failure mode is about the alerting logic being dumb — it's that a single metric crossing a single threshold is a bad proxy for "something is actually wrong."

What you want is something that looks at several weak signals together, decides whether they add up to something worth a human's attention, and — this is the part that matters for anyone who'd have to trust it — leaves behind an exact record of why it decided that.

It's worth remembering how bad the failure mode gets when nothing is watching at all. Knight Capital lost $440 million in 45 minutes on August 1, 2012, because a deployment rolled new code to seven of eight production servers and left an old, dormant order-routing function alive on the eighth — reactivated by a repurposed flag nobody had traced all the way through. Nothing crashed. Nothing errored. It just started sending the wrong orders, correctly, very fast, and no one caught it until the damage was done. That's not a schema-drift alert story, it's several orders of magnitude past one — but it's the same root failure: a silent divergence that no one was independently watching for, in a system where "silent" and "expensive" scale together.

---

## The Shape of It

Three agents, each with a narrow job and a narrower set of tools:

- **Detectors** — one per signal type (schema-registry version history, consumer lag, message-rate volume). Each one only has read access to its own metric. No detector can see another detector's data, and none of them can take any action.
- **Triage** — reads the detectors' output, correlates them, and decides whether this is noise, a low-severity anomaly, or something that needs a human now. It has no tools at all — it only reasons over what the detectors already reported.
- **Narrator** — takes triage's decision and writes the plain-English incident summary a human would actually read. It doesn't get to change the severity or reopen the analysis; its only job is making the record readable.

None of the three can restart a consumer, roll back a schema, or touch the pipeline in any way. That's deliberate. The system's job is to produce a trustworthy claim ("here's what's wrong and why we think so"), not to act on it — acting is still a human decision, same as the EKS post.

---

## What "Scoped" Actually Looks Like

The boundary isn't a prompt telling the agent to behave. It's that the tool doesn't exist. A detector's MCP tool definition looks roughly like this:

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

If an incident actually calls for restarting a consumer or rolling back a schema, that's a different, human-operated system entirely. The monitoring agents were never given a path to it.

---

## The Part That Makes It Auditable

Every step — which detector fired, what evidence it returned, what triage decided, what confidence it assigned — gets written to an append-only log before the narrator ever sees it. Not a summary of the decision. The decision, plus the inputs it was made from.

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

A compliance or risk reviewer replaying this doesn't have to trust the system's judgment on faith. They can see exactly which two signals triage was looking at and why it linked them, and re-derive the same conclusion from the same evidence — or disagree with it, which is the point. The log is the actual audit trail, not a description of one.

---

## What This Isn't

This is a prototype built against a synthetic feed over a few days, not a production system, and I want to be specific about what's still missing before it's one: I haven't measured its false-positive rate over a long time horizon, it hasn't been tested against adversarial or genuinely ambiguous drift patterns, and it has no integration with a real incident-management tool — the narrator's output currently just goes to a log file, not a page.

What I did validate is the part I actually set out to test: that "agents can see everything, but only a narrow, explicit set of tools decide what they can touch" holds up as a pattern outside of coding assistants, and that logging the evidence behind a decision — not just the decision — is what turns "the agent flagged this" into something a skeptical reviewer can actually check.

---

## The Principle, Restated

Same rule as last time, one level up: the interesting design question for an agent isn't how good its reasoning is. It's what it's allowed to touch, and whether the reasoning behind what it flagged is something someone else can independently verify — not just trust.
