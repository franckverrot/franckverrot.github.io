---
title: "The Org That Converges"
date: 2026-08-16 10:45:00
slug: "the-org-that-converges"
tags:
  - ai
  - engineering-leadership
---

Last month I wrote about [convergence engineering](/blog/2026/07/05/convergence-engineering/).  The short version: you declare what the system should be, you watch what it actually is, and the gap between the two becomes the work queue.  Features, bugs, and maintenance all live in that one queue, and the parts that really matter get pinned down with evals written by people who know the domain.  That post was about the mechanism.  This one is about what the mechanism does to the organization running it, because if you take convergence seriously it changes who you hire, what a team looks like, and what a manager is for.  Most orgs adopting AI haven't followed the idea that far.  It leads somewhere uncomfortable.

<!--more-->

## The hiring question

None of this starts with me.  The bluntest version went viral back in spring 2025, when Tobi Lütke's [internal Shopify memo](https://x.com/tobi/status/1909251946235437514) told teams to demonstrate why they cannot get what they want done using AI before asking for more headcount, and Duolingo [said](https://www.cnbc.com/2025/09/17/duolingo-ceo-how-ai-makes-my-employees-more-productive-without-layoffs.html) headcount would only grow if a team could not automate more of its work.  Both memos got read as austerity with extra steps: the Shopify one as a hiring freeze in disguise, the Duolingo one badly enough that users threatened to delete the app and von Ahn spent the next few months [clarifying](https://www.entrepreneur.com/business-news/duolingo-ceo-clarifies-ai-stance-after-backlash-read-memo/492141) that employees weren't being replaced.

I think the cost-cutting reading misses what the question is good for.  Ask "can AI do this job under our harness?" before opening a req and you're forced to locate, precisely, where human judgment is the bottleneck.  If the answer is "the harness could do it but doesn't yet," the req was a request to paper over a factory gap with a salary, and the money is better spent closing the gap.  If the answer is "no harness can do this," congratulations, you've just written the only kind of job description worth writing in 2026.  Every time I've run this exercise without cheating, the bottleneck was almost never more hands to write code.  It was judgment, context, and taste.  Which raises the question of who actually has those.

## The SME-builder

The pipeline we all grew up with: a subject-matter expert knows what good looks like, explains it to a product manager, who writes it down for an engineer, who ships an approximation of an approximation.  Every handoff loses fidelity, and we accepted the loss because building was expensive and experts couldn't build.  That tradeoff is gone.  Building is cheap now, so the highest-leverage person I can imagine is the SME who builds, or the builder who becomes an SME: one person who both knows what good looks like and can put it into the system, as evals, guardrails, and the context the AI builds from.  That's the desired state from Part 1, written directly by the person who knows, instead of translated twice on the way in.

![The handoff pipeline loses fidelity at every step; the SME-builder encodes judgment into the factory, and every team inherits it](/images/sme-builder.svg)

In healthtech, that's the triage nurse who can scan a waiting room and know who can't wait, turning twenty years of pattern recognition into evals the system has to survive: the cases that look mild and aren't.  When one person can't hold both halves (and in healthcare, that's most of the time), you pair them: one SME, one builder, working as a single unit whose output is not features but encoded judgment.  I'd call the pairing a transition state, honestly.  As harnesses mature, the builder half gets absorbed into the tooling, and the SME half is what stays scarce.  That's the part most orgs miss, or don't yet know they'll miss: code generation scales itself, compute scales itself, but what good looks like does not.  The absence shows fast.  Leave architecture undriven and the codebase grows into a monstrosity, one locally reasonable PR at a time; leave design undriven and you ship a pile of screens that don't feel like one product.  Someone has to put the judgment in the machine, domain by domain, decision by decision.

## What happens to teams

If encoded judgment is the scarce asset, team shape follows.

**Small, deliberately.**  Two people share one communication channel, three share three, five share ten, ten share forty-five.  We tolerated the synchronization tax because we needed the hands, and we don't anymore.  Four or five people with a strong harness outship a team of ten without one, and they stay coherent doing it.  (I've tried two and three.  They explode: one departure and the team is gone, and the lead burns out covering the gaps.  There's a floor.  But the ceiling dropped a lot.)

![Two people share one channel, four people share six, ten people share forty-five](/images/communication-channels.svg)

**Expert-generalists over single-stack specialists.**  The iOS-only or backend-only career made sense when every stack cost years to master.  With the model carrying the syntax and the boilerplate, the valuable engineer is what Martin Fowler and his co-authors call the [expert generalist](https://martinfowler.com/articles/expert-generalist.html): deep judgment about systems, applied across whatever stack the work lands in.

![Three specialists, each deep in one layer, versus one expert-generalist: broad across every layer, deep in one or two, standing on the floor the model provides](/images/expert-generalist.svg)

**The platform team ships the standard.**  The deep specialists still exist, they mostly live here now, and the job mutates: less running the platform, more encoding what good looks like into it, so every generalist inherits it for free.  This reframes failure too.  When a bad PR gets through, "an engineer made a mistake" is the wrong incident report; the factory allowed it.  Fix the factory and that PR becomes impossible next week, for everyone.  No mistake paid for twice: not twice by the same engineer, not once each by fifty.

![Product teams stand on a platform team that encodes the standard; an incident fixed in the platform is fixed for every team at once](/images/platform-standard.svg)

## What happens to managers

"I manage 200 people" used to be a career.  Now it's a question: why do you need 200 people?  The manager whose status was headcount is in trouble, and what replaces them is a player-coach: technical, good at building (good, not great, that's fine), whose real skill is connecting dots between teams.  Task distribution goes to the harness.  What's left is the part that was always the actual job: is my team's judgment making it into the system?  Are we picking up what the other teams already learned, or relearning it the expensive way?  Is anyone about to make a mistake the org already paid for once?  A manager who can't read the work can't answer any of those.  That's also why the purely translational management layer, the one whose function was carrying context between other managers, goes first.  The context travels through the system now.

## Amplification has no direction by default

One thing watching real usage data has made obvious: AI didn't turn sloppy engineers into rigorous ones.  It amplifies whoever you already were.  The sloppy engineer ships slop faster, in PRs too large for anyone to review; the rigorous engineer ships rigor faster.  Hand out AI tools without building the factory and you've amplified your variance: your best people got better, your worst got more dangerous, and the gap between the two became your reviewers' problem.  Convergence is what gives the amplification a direction, pulling everyone toward the encoded standard instead of away from it.

(Related: I've resisted productivity leaderboards even while staring at the data weekly.  Measuring generated output rewards exactly the wrong thing.  You get more tokens, and tokens are a cost, not a product.  The people selling you the tokens are, of course, happy to sponsor the confusion.)

## Taste, briefly

None of this stops at engineering.  AI will generate any UI you ask for, and it cannot tell you which one is right, because it has patterns where a good designer has taste, and taste is years of seeing what works, in your domain, for your users.  So the same move applies one more time: designers and PMs stop producing artifacts the system can produce, spend their time with users (which is where the truth was all along), and encode what they learn so the whole system converges on it.  Engineering owns more of the execution.  Product and design own the user.  Both feed the same desired state.

## The junior question

Teams in this picture get smaller, and the people on them read as more senior.  If you're early in your career that sounds like a door closing, so let me be precise about which door.  In [Part 1](/blog/2026/07/05/convergence-engineering/) I said I'm optimistic about what all this does to jobs, and I meant juniors too.  The deep-specialist path still exists; these days it mostly lives on platform teams, where spending years mastering a layer is exactly the job.  On product teams the seniority that matters is domain judgment, knowing the users and why things fail for them, and that one doesn't take a decade to start building: it comes from proximity, and a junior can get proximity on day one.  What did erode is the old on-ramp, learning the craft by grinding through years of implementation work; the model does that grinding now, and the industry hasn't fully replaced what the grind used to teach.  The ladder is still there, it just starts closer to the domain than to the compiler.

## The org it builds

Add all of this up and the result barely resembles the old org with AI tools bolted on.  Its scarce resource is neither headcount nor talent in the abstract: it's judgment that has actually made it into the system.  Every practice in this post is that one idea applied at a different layer, from the hiring bar down to what the platform team ships: scale what your best people know, not how many people you have.  [Part 1](/blog/2026/07/05/convergence-engineering/) was the mechanism, and this is the org it builds.
