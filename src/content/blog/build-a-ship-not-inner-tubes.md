---
title: "Build a Ship, Not a Raft of Inner Tubes"
pubDate: 2026-10-14
description: "Why getting AI into production feels scary, and the one piece of infrastructure that turned a PM's morning idea into a production change by lunch."
tags: ["ai", "healthcare", "infrastructure", "llm", "production"]
---

When you build software, you see a lot of things that are cobbled together, held up with duct tape.

Don't get me wrong, I like duct tape. It works. It's great when you need to patch something up and get it out the door. But you can't really build with it.

At one of my previous jobs we used duct tape, and a lot of it. Prompts were written into the code. Evals were done by hand, with poor metrics. Business logic was spread across several services. Technically it all worked. But it was slow and brittle, and it wasn't something we could monitor or iterate on.

## Inner tubes

Duct-taped AI ends up looking like a raft of inner tubes. Each pilot floats on its own. Each one has to solve deployment, compliance, and monitoring again from scratch. None of them carries much, none of them goes far, and you're never quite sure they'll hold air.

That's a big part of why getting AI into production feels scary. The fear isn't really about the model. It's about having nothing solid underneath it.

## A ship

What you want is a ship. And here's the thing about building a ship: you don't need to know exactly what it'll carry or where it's going. You know three things:

- You're going to go places.
- You're going to need to carry things.
- You don't want to sink.

That's enough to design around. Build in a way that gives you options while staying nimble.

## Containers for what matters

In healthcare, "don't sink" has a specific meaning. If you want to make sure PHI isn't being stored where it shouldn't be, and that you're auditing properly, you need containers to hold things in. You can't guarantee that across a dozen inner tubes. You can guarantee it in one hull.

That's where infrastructure comes in.

## What we built

So we built the ship.

- **A central service to hold our prompts**, instead of having them scattered through the code.
- **A system to evaluate changes**, so we'd know a prompt or model change hadn't made things worse before it shipped.
- **One central path that every prompt ran through**, so access, compliance, and logging happened in one place.

We used open source tools where they made sense: Langfuse for prompt management and tracing, and LiteLLM for routing across models. We deployed to AWS and used both Anthropic and OpenAI models.

But honestly, nothing beats a simple dashboard. Figure out what you want to measure and start measuring it.

For us that was failure rates, first. We made a lot of external calls, and plenty of them were unreliable at times, so we watched how often we had to fall back to another source of data. We also cared a lot about latency. Neither of those is exotic. They're just the questions you can't answer when everything is duct tape.

## What it bought us

Within two weeks we had something we could start piloting, and something we could monitor.

We got that time back in speed almost immediately. Things that used to take days took hours. A PM could come to me with an idea in the morning and it would be live in production shortly after lunch.

One morning a PM asked whether we could add a patient's heart failure risk to the output of one of our prompts. Another service already exposed it through an API, so all I had to do was add it to the prompt as a tool call. CI/CD was already in place, so the change was simple: get an approval, let the tests run, update the prompt. We were live within hours.

In some cases PMs could make changes like that on their own.

All with PHI flowing through it, fully compliant, at low cost, and still nimble.

## Production doesn't have to be scary

This is the part I most want people to hear. Once you have a ship, getting to production stops being a leap. You ship something small that adds value now. You watch how it does, measure it, and improve it. Then you do it again.

The roadblocks that make AI feel stuck are mostly what happens when you have no hull. Build one, and you can move things forward a little at a time, safely.

## Then a fleet

You can start small. One ship, carrying one useful thing. But give it a little time and you'll have an entire fleet.

That first system set us up to start ours. The first thing we shipped on it was a snapshot overview of each patient. Next came a chat agent that let our clinicians ask anything about a patient's history. If the answer was anywhere we had it, whether in the chart, in a transcript of an appointment with a specialist, or in a lab result, the agent found it.

We didn't know we'd build that agent when we started. We didn't need to. We knew we'd go places, carry things, and not want to sink.
