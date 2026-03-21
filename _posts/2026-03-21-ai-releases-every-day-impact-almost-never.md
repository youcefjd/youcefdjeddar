---
layout: post
title: "AI Releases Every Day, Impact Almost Never"
date: 2026-03-21
description: "The gap between what ships and what gets used is still massive — and the hype cycle is starting to catch up."
---

Every single day, there's a new drop.

Anthropic ships something.
OpenAI ships something.
Microsoft ships something.
Some startup claims they just replaced an entire department.

Most recently: an "AI CMO" that allegedly replaces a full marketing team.

Every day, a new revolution.

Every day, less signal.

---

## The Sora Moment (That Nobody Talks About Anymore)

Remember Sora?

The internet certainly does not.

Let's look at the numbers:

* **~$15M/day** compute run-rate
* **~$5.4B/year** in infrastructure
* Pricing: **$1–$5 per 10-second video**

Now the adoption side:

* **~9.6M total downloads**
* Monthly installs: **-32% → -45% decline**
* January installs: ~1.2M

Revenue:

* **~$1.4M total consumer revenue**
* January: **$367K**, down from **$540K** (-32%)

Let that sink in.

Billions in infrastructure.
Millions in hype.
Hundreds of thousands in revenue.

This isn't a failure of technology.

It's a failure of *usefulness*.

---

## The Pattern: Overpromised, Underused

Same cycle, every time:

1. Massive announcement
2. Viral demos
3. "This replaces X industry" takes
4. Brief spike in usage
5. Quiet drop-off

The problem isn't that these tools don't work.

They do.

The problem is that most of them don't matter.

---

## MCP Servers: The Mini Version of the Same Story

Remember when MCP servers were supposed to be the backbone of agentic AI?

The "universal interface." The "missing layer." The thing that would finally make agents useful.

What happened?

### Overpromised value

MCP was supposed to be a silver bullet.

In practice, most servers provide marginal benefit over direct API calls.

Many are thin wrappers with poor abstractions. Disappointing results. Users blaming MCP instead of the implementation.

---

### Developer friction

Setting up MCP in anything beyond a local environment is… painful.

* STDIO vs HTTP confusion
* Secrets, OAuth, session handling
* Crashes, flaky connections, token overhead, latency bloat

It feels premature.

---

### Security reality

Thousands of MCP servers are misconfigured.

Static credentials. Exposed endpoints. Weak access control.

For any serious organization, that's not "cool new infra."

That's an attack surface.

---

### Too much, too mediocre

There are tens of thousands of MCP servers.

Most of them are:

> API → MCP wrappers with extra steps and worse DX

The result is predictable: fatigue.

"Yet another MCP server" doesn't excite anyone anymore.

---

### The hype cycle catches up

People are no longer impressed by demos.

They ask:

* Does this work reliably?
* Does it save time?
* Does it make or save money?

MCP doesn't disappear. It just becomes… quieter.

---

## My Own Usage: Reality Check

I adopted MCP early.

Built servers. Integrated them. Tried to make them part of real workflows.

And then, gradually, I removed most of them.

Replaced with direct API calls.

Simpler. More reliable. Less overhead.

Out of everything in this wave of tools, what do I actually use consistently?

One thing:

> Claude's Excel integration.

That's it.

---

## The Okara "AI CMO" Reality Check

One of the latest examples is the so-called **AI CMO** from Okara — allegedly capable of replacing an entire marketing team.

I actually tried it on a real project.

Not a demo. Not a toy. A real use case:

* Competitor analysis
* Strategy assessment
* Outreach planning
* Positioning recommendations

What I got back felt dated.

Playbooks that might have worked years ago. Generic strategies. Surface-level insights.

Nothing that reflected the reality of 2026.

![Okara AI CMO dashboard — Reddit Opportunities, SEO recommendations, and paywalled Hacker News posts](/assets/images/okara-ai-cmo-dashboard.png)
*The AI CMO feed in action. Reddit mentions and a missing meta description. C-suite material.*

---

### Actual capability vs. marketing

Under the hood, it's not magic.

It's essentially:

> A bundle of agents + LLM + search wrapped in a UI

Which is fine.

What's not fine is presenting that as a **C-level replacement**.

That's not a capability gap. That's a positioning problem.

---

### Product friction (and red flags)

Hands-on usage wasn't smooth either:

* Demos that delivered nothing useful
* Aggressive paywalls
* Broken credit logic (running out while the balance doesn't move)

That's usually a sign of a product that shipped before it was ready.

Hype first. Product second.

---

### Output quality

The outputs tell the real story.

For anything beyond generic content:

* SEO/blog content is surface-level
* Weak in expert domains (fintech, security, etc.)
* Not publish-ready without heavy editing

And more importantly:

> It cannot originate non-obvious ideas.

No creative angles. No bold positioning. No differentiated thinking.

Just execution of obvious playbooks.

It behaves like a **junior assistant**.

Not a CMO.

---

## The Real Problem

We are optimizing for *capability*, not *utility*.

We can generate videos.
We can chain agents.
We can build autonomous workflows.

But the question that matters:

> Who actually needs this, every day, in a way that justifies the cost and complexity?

Most answers are… unclear.

---

## The Illusion of Progress

When you ship something every day, it looks like exponential progress.

In reality, it's often:

* Incremental improvements
* Wrapped in better UX
* Marketed as breakthroughs

Meanwhile, actual adoption concentrates around a few simple, high-leverage use cases.

Not the flashy ones.

The boring ones that work.

---

## Bottom Line

Most AI tools today fall into one of three categories:

* Impressive but impractical
* Useful but overcomplicated
* Actually useful — rare

The gap between *what is possible* and *what is worth doing* is still massive.

And that gap is where most of the hype lives.

---

The industry is shipping faster than it is thinking.

And users are starting to notice.

The winners won't be the teams that ship the most features.

They'll be the ones that build things people actually use — repeatedly, reliably, and without friction.

Until then: expect more daily launches.

And very few things you'll still be using a month later.
