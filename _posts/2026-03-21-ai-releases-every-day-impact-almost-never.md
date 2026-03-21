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

### The permission delegation problem nobody talks about

This is where things get serious.

Most discussions about MCP security focus on exposed endpoints and static credentials — and yes, research shows that **~53% of MCP servers use static API keys**, with only 8.5% implementing OAuth properly.

But the deeper problem is **chained agents and permission propagation**.

Here's a concrete example from my own setup.

I had an MCP server for EKS calling a separate MCP server for CloudWatch to pull logs during incident triage. Simple enough in theory.

In practice:

* What IAM role does the EKS MCP server run under?
* When it calls the CloudWatch MCP server — what permissions does *that* call inherit?
* Is it the EKS role? A new role? The agent's ambient permissions?
* Who validates scope at the boundary between the two servers?

Nobody does.

That's improper token delegation. A token issued for one agent gets passed to another with no scope validation. The receiving server inherits unintended privileges. Access controls break down silently.

The real incident version of this isn't hypothetical: in July 2025, **Replit's AI agent deleted SaaStr's entire production database** — 1,200+ executive records, gone — during an explicit code freeze it was told not to violate. The agent had write access to production it didn't need, executed unauthorized destructive commands, then fabricated data and lied about recovery being impossible. Jason Lemkin recovered it manually from backups. Replit's CEO had to personally respond.

The access controls weren't there. The scope wasn't bounded. And when things went wrong, the agent covered its tracks instead of surfacing the failure.

And this isn't isolated to indie vibe coders.

Amazon mandated AI-assisted development company-wide using their internal tool Kiro. Between December 2025 and March 2026, they hit **four Sev-1 production outages** with AI-assisted code as a contributing factor. The worst: a **6-hour outage on March 5** caused by a deployment that went out without documentation or approval — estimated at **6.3 million lost orders**.

On March 10, Dave Treadwell (who oversees Amazon's retail tech) called a mandatory internal "deep dive" meeting. The internal memo cited "novel GenAI usage" and changes with a "**high blast radius**" as root causes. Their fix: require senior engineers to review all AI-assisted changes to critical systems, and implement "controlled friction."

*Controlled friction.* That's what they're calling it.

That's just… gatekeeping. The thing they were trying to automate away.

In a chained agent setup with multiple MCP servers, you're not just managing one attack surface.

You're managing the intersection of all of them.

And if one server is misconfigured — or one agent has write access it shouldn't — the blast radius isn't contained.

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

## My Own Experience: What I Actually Kept

I adopted MCP early.

Built servers. Integrated them. Tried to make them part of real production workflows.

The EKS + CloudWatch setup I mentioned above? I ran it in prod for a while.

It worked — until it didn't.

Permission delegation was a nightmare. Every time an agent needed to cross a service boundary, I had to manually think through what credentials it had inherited, whether those made sense, and whether I'd accidentally created a privilege escalation path. It wasn't secure by default. It required constant oversight to stay sane.

I eventually stripped most of it back.

Replaced with direct API calls.

Simpler. Auditable. No ambient permission weirdness.

---

Out of everything in this wave of tools, what do I actually use consistently?

One thing: **Claude's Excel integration.**

And honestly — it's genuinely useful. Not in a "cool demo" way. In a "saves me real time every week" way.

I'll admit: I almost learned Excel properly. There was even internal pressure at some point — why aren't you using the full suite, etc.

I resisted. Held the line.

Glad I did.

The integration handles what I need without me becoming an Excel specialist. That's the bar. That's what "actually useful" looks like.

---

## The Okara "AI CMO" Reality Check

One of the latest examples is the so-called **AI CMO** from Okara — allegedly capable of replacing an entire marketing team.

I actually tried it on a real project.

Not a demo. Not a toy. A real use case:

* Competitor analysis
* Strategy assessment
* Outreach planning
* Positioning recommendations

What I got back: outdated playbooks, generic strategies, surface-level insights.

Nothing that reflected the reality of 2026.

One user on HN put it better than I can:

> "Everything it recommended was outdated, super generic, or outright a bad idea."

That's not a one-off. That's the product.

---

### What the dashboard actually shows you

Here's the AI CMO feed from my session:

![Okara AI CMO dashboard showing Reddit Opportunities, SEO recommendations, and paywalled Hacker News suggestions]({{ site.baseurl }}/assets/images/okara-ai-cmo-dashboard.png)

Let's break down what's on screen:

* **Reddit Opportunities** — "2 mentions ready." Paywalled. Upgrade to access.
* **SEO + GEO Recommendations** — "Found 2 issues." The issues? A missing meta description and a performance score of 56. Basic Lighthouse audit stuff.
* **Articles** — AI-written content. Locked behind the Max plan.
* **Hacker News** — Suggested HN posts. Also paywalled.

And elsewhere, the tool literally suggested I post **self-promotional tweets about Okara itself**.

This is the "AI CMO."

Scraped Reddit mentions. A Lighthouse score. Paywalled everything that might actually be interesting.

It's not that the output is wrong. It's that it's completely trivial.

Any junior intern with a browser and a Lighthouse tab could produce this in ten minutes. For free.

---

### Actual capability vs. marketing

Under the hood, it's not magic.

It's essentially:

> A bundle of agents + LLM + search wrapped in a UI

Which is fine.

What's not fine is calling that a **C-level replacement**.

That's not a capability gap. That's a positioning problem.

And it's not unique to Okara. The same pattern shows up everywhere: take a few API calls, wrap them in a dashboard, market it as autonomous intelligence.

---

### Product friction (and red flags)

Hands-on usage wasn't smooth either:

* Demos that delivered nothing actionable
* Aggressive paywalls on every remotely interesting feature
* Broken credit logic — credits running out while the displayed balance doesn't move

That's usually a sign of a product that shipped before it was ready.

Hype first. Product second.

---

### What it can't do

The outputs tell the real story.

For anything beyond generic content:

* SEO/blog output is surface-level
* Weak in expert domains (fintech, security, deep tech)
* Not publish-ready without heavy editing

And more importantly:

> It cannot originate non-obvious ideas.

No creative angles. No bold positioning. No differentiated thinking.

Just execution of obvious playbooks.

It behaves like a **junior assistant with a Lighthouse plugin**.

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
