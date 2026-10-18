---
layout: post
title: "AI Releases Every Day, Impact Almost Never"
date: 2026-03-21
description: "The gap between what ships and what gets used is still massive — and the hype cycle is starting to catch up."
---

There's a new drop every day. Anthropic ships something, OpenAI ships something, Microsoft ships something, and some startup claims it just replaced an entire department. The latest one is an "AI CMO" that supposedly replaces a full marketing team.

The more launches there are, the harder it gets to tell which ones matter.

---

## The Sora Moment (That Nobody Talks About Anymore)

Remember Sora? Most of the internet has already moved on. The numbers are worth a look though.

Cost side:

* **~$15M/day** compute run-rate
* **~$5.4B/year** in infrastructure
* Pricing: **$1–$5 per 10-second video**

Adoption:

* **~9.6M total downloads**
* Monthly installs: **-32% → -45% decline**
* January installs: ~1.2M

Revenue:

* **~$1.4M total consumer revenue**
* January: **$367K**, down from **$540K** (-32%)

So that's billions in infrastructure against hundreds of thousands in revenue. The technology works fine. The problem is that not enough people found it useful enough to keep paying for it.

---

## The Pattern: Overpromised, Underused

The cycle looks the same every time:

1. Massive announcement
2. Viral demos
3. "This replaces X industry" takes
4. Brief spike in usage
5. Quiet drop-off

Most of these tools do work. They just don't end up mattering much to the people who try them.

---

## MCP Servers: The Mini Version of the Same Story

Remember when MCP servers were supposed to be the backbone of agentic AI? The "universal interface," the "missing layer," the thing that would finally make agents useful. Here's how that went.

### Overpromised value

MCP was sold as a silver bullet. In practice most servers give you very little over calling the API directly. A lot of them are thin wrappers with poor abstractions, the results are disappointing, and users end up blaming MCP when the real problem is the implementation.

### Developer friction

Setting up MCP anywhere beyond a local environment is painful:

* STDIO vs HTTP confusion
* Secrets, OAuth, session handling
* Crashes, flaky connections, token overhead, latency bloat

It still feels premature.

### The permission delegation problem nobody talks about

This is the part that worries me.

Most discussions about MCP security focus on exposed endpoints and static credentials, and that's a real issue: research shows **~53% of MCP servers use static API keys**, with only 8.5% implementing OAuth properly. The bigger problem, though, is **chained agents and permission propagation**.

Here's an example from my own setup. I had an MCP server for EKS calling a separate MCP server for CloudWatch to pull logs during incident triage. Simple enough in theory, but in practice:

* What IAM role does the EKS MCP server run under?
* When it calls the CloudWatch MCP server, what permissions does *that* call inherit?
* Is it the EKS role? A new role? The agent's ambient permissions?
* Who validates scope at the boundary between the two servers?

The answer to the last one is nobody. That's improper token delegation: a token issued for one agent gets passed to another with no scope validation, the receiving server inherits privileges it was never meant to have, and access controls break down without anyone noticing.

This has already happened for real. In July 2025, **Replit's AI agent deleted SaaStr's entire production database** (1,200+ executive records) during an explicit code freeze it had been told not to violate. The agent had write access to production that it didn't need, ran unauthorized destructive commands, then fabricated data and claimed recovery was impossible. Jason Lemkin recovered it manually from backups, and Replit's CEO had to respond personally. The access controls weren't there, the scope wasn't bounded, and when things went wrong the agent covered its tracks instead of reporting the failure.

It isn't only indie vibe coders either. Amazon mandated AI-assisted development company-wide with their internal tool Kiro. Between December 2025 and March 2026 they had **four Sev-1 production outages** with AI-assisted code as a contributing factor. The worst was a **6-hour outage on March 5**, caused by a deployment that went out without documentation or approval, estimated at **6.3 million lost orders**.

On March 10, Dave Treadwell (who oversees Amazon's retail tech) called a mandatory internal "deep dive" meeting. The internal memo cited "novel GenAI usage" and changes with a "**high blast radius**" as root causes. The fix was to require senior engineers to review all AI-assisted changes to critical systems and to add "controlled friction." Which, if you squint, is the same human review step the tooling was supposed to remove.

In a chained setup with several MCP servers, you're managing the overlap of all their attack surfaces, not just one. If one server is misconfigured, or one agent has write access it shouldn't, the damage doesn't stay contained.

### Too much, too mediocre

There are tens of thousands of MCP servers, and most of them are:

> API → MCP wrappers with extra steps and worse DX

Unsurprisingly, people are tired of them. "Yet another MCP server" doesn't excite anyone anymore.

### The hype cycle catches up

People aren't impressed by demos anymore. They want to know:

* Does this work reliably?
* Does it save time?
* Does it make or save money?

MCP isn't going away, but it's getting a lot quieter.

---

## My Own Experience: What I Actually Kept

I adopted MCP early. I built servers, integrated them, and tried to make them part of real production workflows. The EKS + CloudWatch setup above ran in prod for a while, and it worked until it didn't.

Permission delegation was the worst part. Every time an agent crossed a service boundary I had to work out by hand what credentials it had inherited, whether they made sense, and whether I'd accidentally opened a privilege escalation path. Nothing about it was secure by default, and it needed constant attention to stay safe.

I eventually stripped most of it out and went back to direct API calls, which are simpler, easier to audit, and don't come with surprise permissions.

Out of everything in this wave of tools, the one thing I use consistently is **Claude's Excel integration**, and it really does save me time every week.

I almost learned Excel properly at one point. There was even some internal pressure to use the full suite. I put it off, and I'm glad I did, because the integration handles what I need without me becoming an Excel specialist. That's what I mean by useful.

---

## The Okara "AI CMO" Reality Check

One of the latest examples is the so-called **AI CMO** from Okara, which supposedly can replace an entire marketing team. I tried it on a real project, not a demo:

* Competitor analysis
* Strategy assessment
* Outreach planning
* Positioning recommendations

What came back was outdated playbooks, generic strategies and surface-level insights, none of it reflecting how things work in 2026. One user on HN put it better than I can:

> "Everything it recommended was outdated, super generic, or outright a bad idea."

From what I saw, that's typical of the product and not a one-off.

### What the dashboard actually shows you

Here's the AI CMO feed from my session:

![Okara AI CMO dashboard showing Reddit Opportunities, SEO recommendations, and paywalled Hacker News suggestions]({{ site.baseurl }}/assets/images/okara-ai-cmo-dashboard.png)

What's on screen:

* **Reddit Opportunities**: "2 mentions ready." Paywalled, upgrade to access.
* **SEO + GEO Recommendations**: "Found 2 issues." The issues are a missing meta description and a performance score of 56, i.e. a basic Lighthouse audit.
* **Articles**: AI-written content, locked behind the Max plan.
* **Hacker News**: suggested HN posts, also paywalled.

Elsewhere, the tool suggested I post **self-promotional tweets about Okara itself**.

So the "AI CMO" is scraped Reddit mentions, a Lighthouse score, and a paywall in front of anything that might be interesting. The output isn't wrong, it's just trivial. A junior intern with a browser and a Lighthouse tab could produce it in ten minutes for free.

### Actual capability vs. marketing

Under the hood it's roughly:

> A bundle of agents + LLM + search wrapped in a UI

That's a perfectly fine thing to build. Calling it a **C-level replacement** is the problem, and it's a marketing problem more than a technical one. Okara isn't unique here either: take a few API calls, put a dashboard on top, and market it as autonomous intelligence.

### Product friction (and red flags)

Using it wasn't smooth:

* Demos that delivered nothing actionable
* Aggressive paywalls on every remotely interesting feature
* Broken credit logic, with credits running out while the displayed balance didn't move

That usually means the product shipped before it was ready.

### What it can't do

For anything beyond generic content:

* SEO/blog output is surface-level
* Weak in expert domains (fintech, security, deep tech)
* Not publish-ready without heavy editing

More importantly:

> It cannot originate non-obvious ideas.

There were no creative angles or differentiated positioning, just the obvious playbooks. It behaves like a **junior assistant with a Lighthouse plugin**, not a CMO.

---

## The Real Problem

We're optimizing for what these tools *can* do rather than whether anyone needs it. We can generate videos, chain agents and build autonomous workflows, but the question that matters is:

> Who actually needs this, every day, in a way that justifies the cost and complexity?

For most of these launches, nobody has a clear answer.

---

## The Illusion of Progress

Shipping something every day looks like exponential progress. Much of it is incremental improvements with nicer UX, marketed as breakthroughs. Meanwhile real adoption ends up concentrated around a few simple, boring use cases that just work.

---

## Bottom Line

Most AI tools today fall into one of three groups:

* Impressive but impractical
* Useful but overcomplicated
* Actually useful (rare)

There's still a big gap between what's possible and what's worth doing, and most of the hype sits in that gap.

The industry is shipping faster than it's thinking, and users are starting to notice. The teams that do well will be the ones building things people use repeatedly and reliably, not the ones shipping the most features. Until then, expect more daily launches, and very few of them still on your machine a month later.
