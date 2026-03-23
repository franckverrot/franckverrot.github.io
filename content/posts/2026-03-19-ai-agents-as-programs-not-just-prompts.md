---
title: "AI Agents as Programs, Not Just Prompts"
date: 2026-03-19 01:48:00
slug: "ai-agents-as-programs-not-just-prompts"
tags:
  - ai
  - programming-languages
---

*Originally posted as a [thread on X](https://x.com/franckverrot/status/2034552496652538359).*

I've been building AI agents for a while now and I'm a bit puzzled about where the industry is going.

We're writing the most autonomous software we've ever built, but the tools we use to build them are a mess. Python script calling APIs, state scattered everywhere, YAML configs, framework-of-the-week. It works until it doesn't, and then you're debugging at 2am because some MCP gateway is stuck as some tool or API changed shape and nothing caught it.

<!--more-->

I started working on something a little bit different: a programming language for agents. There's a thousand languages out there, but nothing that felt right for agents, and that would minimize the learning curve too.

My goal was to be minimalist at first and I ended up near that too: single files for most things, declare models/tools/outside events. All of it is type-checked and spits out native or WASM binaries. The absence of runtime containers, or lengthy node_modules/pip packages is also making this pleasant to use.

I had a lot of feedback from AI engineers about why creating a language was the right idea and while there's probably no real consensus anyway (ie: why compete with python/LangChain?), agents being programs, with state, being crash-free, is appealing and it felt like we needed a new compiler.

Right now, in common frameworks, if an agent doesn't handle a tool error, one would find out in production. If the state of an agent gets mutated somewhere weird, good luck tracing it. If the security team asks "what can this agent actually do?" you're reading through 40 files trying to answer that... so I wanted the compiler to catch that stuff, and report on it.

## The Architecture

For this new language, the architecture is stolen from Elm. There's an `init` function, an `update` function, and a loop. Messages come in: the model responds, a tool returned a result, whatever. The `update` function takes the current state and the message, and returns new state plus what to do next.

It's a bunch of pure functions with no side effects, no hidden mutations, and the compiler checks that you handle every case. The difference from Elm is that here, the messages are things like "the model wants to call three tools" or "the filesystem tool returned an error." And the tools, models, and capabilities are all part of the type system.

```haskell
update msg state =
    match msg with
        GotResponse response ->
            ( { state | phase = "done" }
            , Cmd.done response
            )
        GotToolResult name (Ok result) ->
            ( { state | data = result }
            , Cmd.callModel
            )
```

Anecdotally, I've been trying this on real-life drones, and I'm surprised how well this fits this model of "read signals from the external world, decide, act".

## Where It's At

So today, the compiler is real, there's a LSP for editor support, a linter/formatter, and many more useful things for engineers to play with (there's even mDNS agent-to-agent comms.)

LLVM is great for that too, binaries are fast and small.

I still love Python for quick prototypes. If you're hacking something together for a demo, just use LangChain or whatever. The learning curve for Haskell or Elm-like languages is not steep but not worth investing if you're learning all of these new-ish AI concepts.

But... if you're building agents that run in production, that need to be auditable, that other people on your team need to understand and trust... I think there's a gap. And I think a language fills it better than another framework... More soon. Hopefully...

I want to try building this in the open but don't want to release AI slop, so there's more testing to be done too. I'll report on it when I stop replacing propellers for a living.

If any of this resonates, follow along. And if you think it's a terrible idea, I want to hear that too.
