---
layout: post
title: "Stop Asking Your Agents to Be Neutral"
date: 2026-10-17
description: "You can't prompt an agent into neutrality. So build the system so nobody has to be neutral, and let something you actually observed cast the deciding vote."
---

[Two weeks ago]({% link _posts/2026-10-03-naming-a-bias-is-not-escaping-it.md %}) I watched a model name its own bias and then walk straight into it. Same conversation. Sometimes the same paragraph. So the obvious next question: if you're building a system out of these things, what do you actually do about it?

The popular answer is a better prompt. "Be neutral." "Be critical." "Be objective." I've tried all of them. They don't work, and I've stopped pretending they will.

What works is boring. Design the system so no agent ever has to be neutral, because the deciding vote comes from something you observed, not something a model said.

---

## Why "Be Neutral" Doesn't Work

A model's answer drifts toward whatever its framing predicts. Ask for a verdict in a hopeful setup and you get a hopeful verdict. Ask in a doubtful one and you get doubt. The evidence underneath didn't move an inch. Only the mood did.

Telling it to be neutral just swaps in a different mood. Neutral is a framing too, so you get back what a neutral-sounding agent is supposed to sound like. The costume of impartiality. Same with "be cautious" or "be humble." It can do the voice. That doesn't mean anything is behind the voice.

The skeptic judge is my favourite version of this, because it looks so much like a fix. You add a second agent and tell it to be critical. Does it evaluate the first agent's reasoning? Of course not. It produces the critical response the prompt asked for. Congratulations: you've traded false confidence for false doubt, and the system now rejects correct diagnoses just as smoothly as it used to accept wrong ones.

And no, throwing more models at it doesn't help. [Ensembles](https://en.wikipedia.org/wiki/Ensemble_learning) reduce error when the errors are independent. These aren't. Every model in the chain reads the same framing and leans the same way. Three agents agreeing off the same setup is one opinion said three times, with bonus confidence for the repetition.

---

## The Loop

One rule: nothing moves forward because an agent said so.

![Flowchart of the loop: agent names a cause and a probe, the probe and a pre-set signal decide, failures auto-revert, humans only see irreversible or ambiguous cases]({{ site.baseurl }}/assets/images/agent-loop-deciding-vote.svg)

The probe answers the first question. A signal you picked in advance answers the last one. If an action fails, it reverts itself before anyone gets a chance to argue about why.

Three moves get you there.

---

## Move 1: Every Diagnosis Comes With a Probe

An agent doesn't get to act on its diagnosis. First it names a probe: a test whose possible results tell the candidate causes apart. Not an argument for its favourite theory. A test. The agent names it, the system runs it, the result decides.

The probe has to actually discriminate. If it comes back the same no matter which hypothesis is true, it's confirmation theatre. Feels like evidence, carries none. So before it runs, each candidate cause has to predict a different outcome.

| Hypothesis | Probe | What you'd see if it's true |
|---|---|---|
| Bad config shipped | Diff the live config against the last known-good revision | Non-empty diff on the suspect key |
| One bad replica | Pin traffic to each replica in turn, 60s each | Error rate follows one replica |
| Upstream dependency degraded | Pull the dependency's own latency metric for the window | Their p99 climbs before ours does |
| Load-driven saturation | Compare the incident window to the same hour last week | Request volume outside the normal band |

Two rules keep this honest. The probe gets written before anyone sees the result, so the agent can't reverse-engineer a test that confirms what it already likes. And if nobody can come up with a probe that separates the hypotheses, that's the answer. The agent reports "ambiguous" instead of picking whichever cause sounds most plausible.

The probe result is the only thing in the whole loop that no model wrote. Everything else is a model reacting to a framing.

---

## Move 2: Make It Reversible, Let the Result Judge

Sometimes a probe is too expensive, or there just isn't one. Fine. Make the action the probe. Restart the pod behind something you can roll back. Shift 5% of traffic. Flip the flag for one tenant. Then watch the signal. Production is a far more honest critic than any model, mostly because it never read your prompt.

This turns reliability into an infrastructure problem instead of a prompting problem, which is great news, because we already know how to solve infrastructure problems. [Feature flags](https://martinfowler.com/articles/feature-toggles.html) with instant rollback, [canary routing](https://martinfowler.com/bliki/CanaryRelease.html), saving the old state before you write the new one, idempotent operations, [blue-green](https://martinfowler.com/bliki/BlueGreenDeployment.html) at the model-routing layer. Everything you already use to deploy safely doubles as a way to check an agent's claim.

For the result to count, two things get decided up front:

- **The success signal, before the action.** Which metric, which direction, what window. Let an agent read a dashboard after the fact and it will find a story in there. Every time. Give it a threshold it committed to beforehand and there's no story to find.
- **The rollback trigger, automatic.** The revert fires on the signal, not on the agent's reading of the signal. Otherwise you've quietly handed the decision back to model judgment, right when it matters most.

The nice part is most of the loop stops needing to be right. An agent that's wrong 30% of the time but whose mistakes roll back in 90 seconds off a metric is something I can use. An agent that's wrong 5% of the time with no undo button is not.

---

## Move 3: Show It Evidence, Not Arguments

Sometimes a second model really is the only option. Fine. Just don't show it the first model's reasoning. Hand it the logs, traces and metrics cold, no story attached, and see where it lands.

That's the entire value of the step:

- If it agrees after reading the first agent's argument, that's contagion. It's finishing a text whose conclusion is already on the page.
- If it agrees from the raw evidence alone, that's weak but real independent support. Not fully independent, shared pretraining sees to that, but you've removed the biggest shared input, which is the framing.

So the orchestrator decides what the reviewer sees, not the agent. In practice I split the first agent's output in two. The conclusion and recommended action go to the executor. The reasoning goes to a human-readable log and nowhere else. The review call gets built from the evidence bundle, never from the conversation.

Same for retries. Re-prompting a failed agent inside its own context isn't a second attempt. It's the same attempt, now under pressure to stay consistent with itself. If a step needs a retry, rebuild the context from the evidence and start fresh.

---

## Where Model Judgment Is Still Hiding

The audit is mechanical. Find every spot where one model's output feeds another model's assessment, and ask whether an observation could take its place.

| Pattern | What you actually get | Replace it with |
|---|---|---|
| Judge model scores another agent's reasoning | Whatever its framing calls for | A probe result, or a scored run on held-out cases |
| "Critique this plan" review step | Fluent objections, whatever the plan's quality | Run it in a sandbox with assertions |
| Agent self-reports a confidence score | A number shaped by the prompt's tone | That agent's actual track record on that task type |
| Majority vote across three agents, same context | One opinion counted three times | Independent contexts, or drop the vote |
| Agent summarises its tool output for the next agent | A lossy retelling that locks in one interpretation | Pass the raw output; summarise for humans only |
| Retry with "think more carefully" | Pressure to repeat the first answer | A fresh context rebuilt from the evidence |

What's left after this is a small, honest job for the human. Not reviewing every action, because that's how you burn someone out on cases the system could've settled on its own. The human belongs at the forks that are irreversible or unobservable: the migration you can't roll back, the message you can't unsend, the change whose effect won't show up in any signal you collect. Everything else goes to the probe or the revert.

Two warning signs once it's running. If an agent finds a probe for every single hypothesis, it's probably producing probe-shaped text rather than real tests, so sample a few and check the outcomes actually separate. And if escalations to humans keep dropping, hold the champagne. That's just as likely to mean you've misclassified what's irreversible as it is to mean things are getting better.
