---
title: "Convergence Engineering"
date: 2026-07-05 15:30:00
slug: "convergence-engineering"
tags:
  - ai
  - engineering-leadership
---

Everyone is writing about loops right now.  Boris Cherny [said he stopped prompting Claude](https://thenewstack.io/loop-engineering/): loops prompt Claude now, and his job is to write the loops.  Andrew Ng [followed up with three nested ones](https://x.com/AndrewYNg/status/2071988145667928442), an agent coding loop inside a developer feedback loop inside an external feedback loop, Russian dolls all the way down.  "Loop engineering" is the buzzphrase of the summer, and the takes are multiplying.

I've been chewing on this for a few weeks and I think the loop is the wrong thing to name.  We named continuous integration after its mechanism too, but what made CI matter was the invariant it maintained: a codebase that never drifts far from an integrable state.  The loop was just how you got there.  I think the same thing is happening again, one level up, and I've started calling the actual discipline convergence engineering.

<!--more-->

## Drift is the work queue

Here's the model I keep coming back to.  Every system we ship has two descriptions that never stop diverging: what it should be (specs, docs, architecture decisions, the pitch deck, the stuff in people's heads) and what it actually is (the code, the running system, the telemetry.)  For my whole career, keeping those two in sync was a chore nobody did.  Engineers hated writing docs because the code was the only artifact that couldn't lie, so why maintain the ones that could?

The bet I'm making is that this reconciliation stops being a chore and becomes the mechanism development runs on.  Declare a desired state.  Observe the actual state.  Diff them.  The diff is drift, and drift items are the work queue.  Agents work the queue.

![The convergence loop: desired state and observed state feed a diff, drift items become the work queue, agents work the queue](/images/convergence-loop.svg)

Once you see it this way, a lot of things collapse into one thing.  A new feature is drift you introduce on purpose: you edit the desired state, and the diff against the observed system becomes the backlog.  A bug is drift you didn't ask for, arriving from the other side.  A refactor, a doc update, a dependency bump: same queue.  There's no separate "maintenance loop" bolted onto the side, because maintenance and development are the same operation pointed at different diffs.

If you've run infrastructure this should feel familiar, because it's the Kubernetes controller pattern applied to software engineering itself.  And that comparison is exactly why I keep insisting convergence is the mechanism, not the goal.  Nobody running Kubernetes "drives toward" convergence at quarterly milestones.  The reconciler runs, and converging is simply what the system does.  The goal lives in the desired state.  The loops (and there will be many, nested, boring) are the actuator's implementation detail.

## The pinned surface

The obvious objection: English is ambiguous, so diffing a prose spec against a codebase produces opinions, not deltas.  True, if you try to specify everything: so don't.

Kubernetes works partly because of what it refuses to make you specify.  You declare replica counts; you never declare pod IPs.  Omitted fields are degrees of freedom you've granted the system.  I think the desired state for software wants the same shape: deliberately partial.  Pin the things that matter, make those unambiguous (unambiguous enough to evaluate, at least), and let everything else float.  Divergence outside the pinned surface is just the agent's latitude, and nobody adjudicates it.  Drift only exists where something is pinned.

![The pinned surface sits inside a larger free zone; items move from the free zone into the pinned surface when they start to matter](/images/pinned-surface.svg)

A product maturing *is* its pinned surface growing.  Something lives in the free zone until it starts to matter (an incident touches it, a customer depends on it, a decision gets made), and then you pin it.  Which means the engineering judgment call of this era is deciding what to pin and when:
* Pin too little: the system drifts where it hurts.
* Pin too much: you've rebuilt waterfall with extra GPU spend.

## Evals are how you pin, and experts write them

A pin is worthless if nothing can observe whether the system honors it.  Code has compilers, type checkers, and CI.  A prior authorization policy has none of those.  The sensor for everything else is an eval, which gets me to the definition I like most in this whole framing: the pinned surface is exactly the set of things someone wrote an eval for.  If nobody can evaluate it, it isn't pinned, whatever the document says.

So who writes the evals?  People with taste, business context, and deep expertise, because writing a good eval takes exactly the three things models don't have.  I'll take my examples from healthtech, since that's where I've spent years of my career.  The clinical expert stops hand-reviewing eligibility decisions and writes the evals the eligibility engine must survive: synthetic members, the edge cases behind real denials, the rules that change every plan year.  The compliance officer stops eyeballing screens before an audit and encodes what PHI handling must never do.  The security expert stops writing auth code and writes the evals auth code must survive.  Same move for architecture, tests, business docs, the pitch to your first payer.  Ng calls the human contribution a context advantage rather than taste, and I think evals are what that advantage looks like once you make it durable: tacit context turned into executable judgment.

That's also my bet on what a company looks like in a few years: a set of pinned surfaces and eval suites, each owned by an expert, with reconcilers running against them.  The org chart starts to mirror the ontology.  Experts mostly stop producing the artifacts and instead ensure the system produces resilient ones.  And notice this widens who gets to build: a domain expert who can write evals is a first-class author of the desired state, no engineer needed as translator.

I know how the labor half of this sounds given the apocalyptic mood out there, so let me be precise about why I land on the optimistic side.  Generation is what automates.  Judgment is what concentrates.  The eval is the form judgment takes when it has to scale, and it needs accountable humans behind it (more on that regress below.)  That's a bright picture for people who know things.  The transition will be genuinely rough for pure-implementation roles caught in between, and I won't pretend otherwise.  But "no more jobs" gets the direction wrong: the constraint moves from producing artifacts to knowing what good ones look like, and that's people, for a long time.

## We've seen this movie before, one layer down

If you're skeptical this can reach the application layer, fair, but look at the pattern.  Declarative diff-and-apply has won every layer where the diff got cheap: Terraform for infra, Kubernetes for orchestration, GitOps for deployment, React for UIs, declarative migration tools for schemas.  It never climbed up to software itself for one reason: you couldn't diff English intent against a codebase and a running system.  LLMs are the first plausible diff engine for that pair.

And we did try.  Model-driven architecture, UML round-tripping, executable specifications: same dream, twenty years earlier, dead because both the diff and the apply required humans or brittle codegen.  The architecture was right and the economics were wrong.  The economics are what flipped.

Most of the "AI software factory" products I've looked at still get this backwards, by the way.  They're pipelines: requirements in, an ontology in the middle, code compiled out the other end, maintenance bolted on after.  I reviewed one [back in March](/blog/2026/03/22/build-vs-buy-your-software-factory-an-8090-review/) and my complaint at the time was that automatic drift detection between docs and code was the thing that would have made it worth paying for, and it didn't ship.  In the controller picture, generation is just the actuator.  Drift detection is the product.

## Tests and specs still kicking in 2026 and onward

Two obituaries I keep reading and don't buy.

Test-Driven Development as a human ritual (red, green, refactor, by hand) is fading, sure.  But in the controller picture, tests get a promotion: they stop being your design tool and become sensors, part of the observation function that makes the system's actual behavior readable at all.  Same for traces and monitoring: the investment moves from writing the code to instrumenting the truth.

Spec-driven development has the same shape.  A frozen upfront spec is short-sighted, and the practical limits are real: models only follow so many standing instructions before compliance degrades, so a thousand-line spec file is mostly decoration.  But a spec that's a living desired-state declaration, kept small by the pinned surface and partially generated by the drift detector itself, is a different animal.  When the observed system does something the spec never mentioned and users love it, the drift resolves by updating the spec, not the code.  Reconciliation runs in both directions, and that's exactly where humans sit in this: adjudicating which side of the diff is truth.  A much more precise job description than "human in the loop," if you ask me.

## The hard parts

Where the real work is in my opinion:

**Observation is partial.**  Kubernetes can read its entire observed state out of `etcd`.  You can't read a software system's behavior out of anything; tests, evals, and telemetry are all partial sensors.  The diff is only as good as the instrumentation, so eval coverage becomes the thing you argue about in planning meetings, the way test coverage used to be.

**Control loops oscillate.**  An agent fixing drift can introduce new drift, forever, and two reconcilers can fight each other politely until the money runs out.  You need damping: budgets, circuit breakers, oscillation detection.  The boring control-theory toolbox, basically.

**Evals rot too.**  They're artifacts in the same mesh as everything else, so they drift and need reconciliation like the rest.  And expert review has its own failure mode: rubber-stamping, which is the reviewer's version of what Addy Osmani [calls cognitive surrender](https://addyosmani.com/blog/loop-engineering/).  Yes, there's a who-evaluates-the-evaluators regress in here.  It bottoms out at a human with accountability, which is precisely why the jobs don't disappear.

## The dream pass

One more piece, and it's the one that convinced me the framing holds together.  Karpathy has been pushing the idea of an LLM-maintained wiki as a second brain: the model owns the pages, links concepts, and runs health checks that catch contradictions and staleness.  Anthropic shipped memory consolidation along similar lines this spring, and there's a [whole lineage](https://ogham-mcp.dev/blog/memory-consolidation-lineage/) behind the idea going back to how sleep consolidates biological memory.  I built a tiny version of this into my own vault earlier this year: a `dream` pass that sweeps for stale facts and contradictions and proposes fixes.

In the convergence framing, the dream pass is reconciliation applied to the intent artifacts themselves: specs, decisions, evals, all periodically consolidated so the desired state doesn't rot while the loop runs against it.  It's the damping term, every artifact pair needs one, and right now almost nobody is building them.

## What I'm doing about it

Building it.  I won't say much about it yet, but it's a software factory organized around the controller picture instead of the pipeline one.  Drift detection at the center, generation as the actuator, evals as the pinning mechanism.  It's early, and I'm obviously biased, but it already behaves differently from every factory product I've test-driven.  The property that keeps striking me is that it accelerates as it fills up: every spec, decision, and eval thrown into it makes the next diff sharper and the work queue better ordered, so the more organized it gets, the faster it moves.  Pipelines clog as you feed them.  This does the opposite, and that difference is the whole bet.

The loops are real, and I'll keep writing them.  I just don't think they're the discipline.  Convergence is.
