---
title: "Build vs. Buy Your Software Factory: an 8090 Review"
date: 2026-03-22 11:13:00
slug: "build-vs-buy-your-software-factory-an-8090-review"
tags:
  - ai
  - engineering-leadership
---

*Originally posted as a [thread on X](https://x.com/franckverrot/status/2035781987630539064).*

I was reading that 8090 just landed a partnership with EY last week, and this got me interested in test-driving it. Most of the coverage out there focuses on enterprise usage, I wanted to know if it could help me on an open source project (no budget, PMs, just me/Claude/my repos.) And if this is good for thousands of consultants, maybe it could also be good for the day job.

I'll start by saying that I liked the pitch: the bottleneck in (a lot of, not all obviously) software is deciding what to build, not writing the code. 8090 wants to be the single source of truth that connects product decisions to engineering execution: a gap that's more or less narrow in various companies, and that doesn't exist in side projects (where I'm the CEO/CFO/engineer/QA/product marketing person all at once.)

<!--more-->

## How it works

Four components in the software factory:

- **Refinery** is where you write your PRD.
- **Foundry** turns that into engineering blueprints.
- **Planner** generates codebase-aware "work orders."
- **Validator** turns user feedback into dev tasks.

They build a "Knowledge Graph" that connects requirements, architecture, and implementation. I find it really intuitive: when something changes upstream, the downstream components seem to update automatically.

## Easy setup, one friction

GitHub integration installed fine. After installing the app and granting repo access (only one repo), I had to manually add my repository again. Wish a simple dropdown was there but honestly I'm nitpicking at this point... or am I? It's a small thing, but it's the first hint I got to start taking a perspective that the whole tool is built to suit their workflow, not mine.

## Generating requirements is great

The Refinery module was genuinely useful. I've been able to describe what I was building, it's been able to generate knowledge from the codebase, and generate structured feature requirements that really made sense. The info hierarchy is solid and reasonable, the sort of info I would actually have in my own repo nowadays without this service.

But... it should have happened during onboarding. Software Factory already has access to my codebase, and could have auto-generated a product overview: "here's what we think your project does today." That would have saved me the work of describing what already exists, and more importantly, it gives me an honest assessment of how far my project is from what 8090 considers well-structured software.

They clearly have opinions about what good looks like and I was coming there to find exactly that, and I would have understood why I was paying the $200/mo plus AI usage on top on that: paying for my own utilization is fine, but why not use my first month payment to do a thorough analysis of the project, and show me how to "get started with their unique subject matter expertise." They publish writing guides for requirements, blueprints, and work orders: show me the gap. That would demonstrate the value immediately instead of asking me to take it on faith.

## Work orders only for net new work

Planner generates work orders from blueprints. For new features, it's solid. It knows your codebase, tells you which files to touch, describes the work. I haven't gone that far this time around with my POC, but I've seen this in action on a call with their sales team some time ago (which, btw, are incredibly great at demonstrating the value, kudos to them!)

Problem: it's not meant to look backward. I wished there was a way to build a changelog view from commits, get a "here's how your codebase evolved and what decisions are embedded in it, and the shortcomings/great things you built so far."

For me, especially with OSS I modify only from time to time, the hard part isn't planning the next feature, it's understanding the accumulated decisions that shaped what's already there (and whether my past decisions were deliberate or just expedient.) 8090 mentions "reverse engineering" for legacy modernization on the enterprise side, so the capability probably exists somewhere, though... It just doesn't show up in the standard product or I totally missed it.

## The real question

Software Factory gives you structure, a framework for thinking through requirements, architecture, and implementation in an organized way. I have a strong personal bias that this matters a lot, always tried to push in this direction: build ADRs, remove tribal knowledge, enable all of Eng/Product/Design all at once by building more documentation that makes sense.

But I kept asking: what does this solve that a bunch of well-crafted prompts and a disciplined docs habit wouldn't? A system prompt that enforces PRD structure. A set of scripts that maintains architectural context. A prompt chain that generates implementation plans from specs. You could also back-document existing work that way, which Software Factory currently doesn't do (or push for... again, I haven't pushed more, there's maybe a way but it seems counter to their ways of working.) Put that all together and you've basically built your own software factory. It's just not a product or platform.

The unique value of a platform like this should be the stuff you can't easily replicate with prompts: persistent knowledge graphs that evolve with your codebase, automatic drift detection between docs and code, retrospective analysis of past decisions. Some of that is in 8090's roadmap, but not all of it ships today.

## Who this is for

Definitely Enterprise teams, where there's a strong need for governed, documented, and auditable development workflows. The EY deal makes total sense for that. If you're deploying thousands of people to build and modernize client software, a structured orchestration layer that keeps everyone aligned is worth paying for.

For open source or solo work, it's either early or unnecessary (or I should have spent more time on it maybe.) I think the thinking is right and our industry needs better context engineering upstream of code generation, but the product assumes you're starting fresh, not making sense of something that already exists. And it doesn't yet justify the structure over a thoughtful prompt setup.

## Honest take

8090 is betting that better software comes from better decisions before code gets written, not from faster code generation: love the bet, it's reasonable and the platform has no hiccups.

To get there for more than enterprise teams, I think it needs to:

- auto-generate the initial product overview from existing code
- support looking backward not just forward
- and show me concretely why its structure beats a prompt library

The framework is solid, and now it needs to prove it's worth the extra platform and associated spend.
