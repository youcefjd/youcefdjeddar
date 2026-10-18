---
layout: post
title: "Stop Asking Your Agents to Be Neutral"
date: 2026-10-17
description: "You can't prompt an agent into neutrality. So build the system so nobody has to be neutral, and let something you actually observed cast the deciding vote."
---

[Two weeks ago]({% link _posts/2026-10-03-naming-a-bias-is-not-escaping-it.md %}) I watched a model name its own bias and then walk straight into it, sometimes in the same paragraph. If you're building a system out of these things, that's a problem you have to design around.

Most people try to fix it in the prompt. "Be neutral," "be critical," "be objective." I tried all three and none of them held up.

What I do now is set things up so no agent ever has to be neutral in the first place. The final call comes from something I can observe in the system, like a metric or a test result, and never from a model's opinion.

---

## Why "Be Neutral" Doesn't Work

A model's answer drifts toward whatever the framing suggests. Set up the question hopefully and you get a hopeful verdict, set it up doubtfully and you get a doubtful one, even when the evidence is identical.

Asking it to be neutral doesn't get you out of that, because "neutral" is a framing as well. The model gives you what it thinks a neutral agent sounds like. Same story with "be cautious" or "be humble": it can copy the tone without any of the judgment you were hoping would come with it.

The skeptic judge is the version that fools people most, because it looks like a fix. You add a second agent and tell it to be critical, and it doesn't really check the first agent's reasoning. It writes the critical response you asked for. So you've traded false confidence for false doubt, and now the system rejects correct diagnoses as easily as it used to accept wrong ones.

Adding more models doesn't help either. [Ensembles](https://en.wikipedia.org/wiki/Ensemble_learning) reduce error when the errors are independent, and these aren't: every model in the chain reads the same framing and leans the same way. Three agents agreeing off the same setup is basically one opinion counted three times, and it looks more confident than it should.

---

## The Loop

The basic rule is that nothing moves forward just because an agent said so.

![Flowchart of the loop: agent names a cause and a probe, the probe and a pre-set signal decide, failures auto-revert, humans only see irreversible or ambiguous cases]({{ site.baseurl }}/assets/images/agent-loop-deciding-vote.svg)

A probe decides whether the diagnosis holds up, a signal you chose before acting decides whether the fix worked, and a failed action reverts on its own without anyone debating it.

Here's how each part works.

---

## Move 1: Every Diagnosis Comes With a Probe

An agent can't act on its diagnosis directly. It first has to name a probe, meaning a test whose possible results would tell the candidate causes apart. The agent proposes the test, the system runs it, and the result decides what happens next.

The probe has to actually discriminate between causes. If it comes back the same whichever hypothesis is true, you've learned nothing, even though it feels like you checked something. So before it runs, each candidate cause should predict a different outcome.

| Hypothesis | Probe | What you'd see if it's true |
|---|---|---|
| Bad config shipped | Diff the live config against the last known-good revision | Non-empty diff on the suspect key |
| One bad replica | Pin traffic to each replica in turn, 60s each | Error rate follows one replica |
| Upstream dependency degraded | Pull the dependency's own latency metric for the window | Their p99 climbs before ours does |
| Load-driven saturation | Compare the incident window to the same hour last week | Request volume outside the normal band |

I hold to two rules here. The probe is written before anyone sees a result, so the agent can't design a test around the answer it already likes. And if nobody can come up with a probe that separates the hypotheses, the agent reports the situation as ambiguous rather than going with whichever cause sounds most plausible.

The probe result is the one input in the loop that no model produced. Everything else has gone through a model at some point.

---

## Move 2: Make It Reversible, Let the Result Judge

Sometimes a probe is too expensive or there isn't one. In that case the action itself can be the test: restart the pod behind something you can roll back, shift 5% of traffic, or flip the flag for one tenant, then watch the signal. Production doesn't care how the prompt was worded, which already makes it a better judge than another model.

This makes reliability mostly an infrastructure problem, and we have plenty of tools for those: [feature flags](https://martinfowler.com/articles/feature-toggles.html) with instant rollback, [canary routing](https://martinfowler.com/bliki/CanaryRelease.html), saving the old state before writing the new one, idempotent operations, and [blue-green](https://martinfowler.com/bliki/BlueGreenDeployment.html) at the model-routing layer. Whatever you already use to deploy safely can also be used to check an agent's claim.

Two things need to be decided before the action for the result to mean anything:

- **The success signal.** Which metric, which direction, over what window. If an agent reads the dashboard after the fact, it will usually find some way to read it as success. A threshold it committed to beforehand doesn't leave that room.
- **An automatic rollback trigger.** The revert should fire on the signal itself, not on the agent's interpretation of it. Otherwise the decision ends up back with the model, at exactly the point where you least want that.

Once this is in place, the agent doesn't need to be right very often. I'd take an agent that's wrong 30% of the time with mistakes that roll back in 90 seconds over one that's wrong 5% of the time with no way to undo anything.

---

## Move 3: Show It Evidence, Not Arguments

Sometimes a second model really is your only option. If so, don't show it the first model's reasoning. Give it the logs, traces and metrics with no explanation attached and see what it concludes.

How it got to agreement matters a lot here:

- If it agrees after reading the first agent's argument, that's not worth much. It's continuing a text where the conclusion was already written.
- If it agrees from the raw evidence alone, that's some independent support. Shared pretraining means it isn't fully independent, but you've at least taken away the framing, which is the biggest thing they'd otherwise share.

That means the orchestrator, not the agent, controls what the reviewer sees. In practice I split the first agent's output into two parts. The conclusion and recommended action go to whatever executes it, and the reasoning goes into a log for humans and nowhere else. The review call is built from the evidence, not from the conversation.

Retries work the same way. Re-prompting a failed agent in its own context mostly gets you the same answer again, because now it's also trying to stay consistent with what it already said. If a step needs a retry, rebuild the context from the evidence and run it fresh.

---

## Where Model Judgment Is Still Hiding

To find the rest, go through the design and look for every place where one model's output feeds into another model's assessment. For each one, ask whether you could use an observation of the system instead.

| Pattern | What you actually get | Replace it with |
|---|---|---|
| Judge model scores another agent's reasoning | Whatever its framing calls for | A probe result, or a scored run on held-out cases |
| "Critique this plan" review step | Confident-sounding objections, regardless of the plan | Run it in a sandbox with assertions |
| Agent self-reports a confidence score | A number shaped by the prompt's tone | That agent's actual track record on that task type |
| Majority vote across three agents, same context | One opinion counted three times | Independent contexts, or drop the vote |
| Agent summarises its tool output for the next agent | A lossy retelling that locks in one interpretation | Pass the raw output; summarise for humans only |
| Retry with "think more carefully" | Pressure to repeat the first answer | A fresh context rebuilt from the evidence |

After all that, the human's job gets a lot smaller. You don't want someone reviewing every action, since that wears people out on cases the system could have handled itself. Humans should be at the decisions that can't be undone or can't be observed: the migration with no rollback, the message that can't be unsent, the change that won't show up in any signal you collect. The probe or the revert handles everything else.

Once it's running, I watch for two things. If an agent always manages to find a probe for every hypothesis, it's probably writing things that look like probes, so sample a few and check that the outcomes really differ. And if escalations to humans keep going down, don't assume things are improving. It's just as likely that something irreversible has been classified as reversible.
