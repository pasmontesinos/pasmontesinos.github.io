---
title: "Where I put my focus when I delegate to agents"
slug: "where-i-focus-when-delegating-to-agents"
translationKey: "where-i-focus-when-delegating-to-agents"
date: 2026-10-13T08:00:00+02:00
draft: false
tags:
  - AI-Assisted-Development
  - Agents
  - LLM
  - TDD
  - Continuous-Delivery
  - Lean
  - XP
categories:
  - Development
  - AI
description: "How I went from steering agents in real time to a loop where I decide the spec and the merge, and the agent works by my rules."
---

Almost a year ago I wrote about [how to keep the discipline in AI-assisted TDD](https://pasmontesinos.com/en/posts/ai-assisted-tdd-flow/). In that flow I steered the agent in real time, in the same session where the code was being written: I picked every test, corrected the design on the fly and approved every commit.

That flow changed gradually. Models got better from one version to the next, and the repository conventions got sharper with every correction I wrote down. With both, it made less and less sense to steer the implementation step by step, and more sense to put the weight on the spec: make clear what to build and how, and delegate the rest. At some point the implementation was running on autopilot until it was ready for review.

But I was still in the middle, jumping from one Claude Code session to the next. Hand over the spec, kick off the implementation, ask for the review, apply the comments, start the next slice. All of it through repetitive prompts, session after session.

[grooveloop](https://grooveloop.pasmontesinos.com) was born to replace that coordination. And once it existed, other things came almost on their own: automatic correction rounds, independent slices in parallel, and review lessons folded back into the conventions.

| | Before | Now |
| --- | --- | --- |
| Design decisions | In real time, while the agent writes | In the spec, before starting |
| Corrections | On the fly, in the session | In the review, and turned into conventions |
| Implementation | Steered step by step | On autopilot up to the PR |
| Coordination between steps | Me, prompt by prompt | The loop |

This post is not about the tool, but about **where my attention has ended up** after that change. It is not a recipe or a productivity promise. It is what is working for me, with its limits. If you already work with agents but still steer them session by session, it might help you take the next step: delegate more without losing control, and with less friction.

## The flow, in one picture

![The cycle of a slice: brainstorming and spec, sub-issue, developer opens PR, automatic reviewer, my review, next slice or conventions PR](/images/posts/grooveloop-flow.en.svg)

The starting point is a Story that has already been thought through and cut into slices; how I get there I explain below. From there the loop creates one sub-issue per slice and launches an agent that implements it in an isolated worktree and opens a PR. Another agent reviews it before I do. I decide: merge, close or ask for another round. The merge unblocks the next slices, and the independent ones move forward in parallel.

## Where I put the emphasis

My attention used to be inside the session, in real time. Now it is at the ends: in the spec and the conventions, before the agent starts, and in the review, before anything reaches master. In between, autopilot.

### 1. The spec is the prompt

The flow does not start by writing a spec, but by talking. I start from a user story or a task and open a brainstorming session with the agent. There we dig into how to approach the change: architecture, design, delivery strategy and any doubt that could end up as a misunderstanding.

Only when that is clear do I write the Story. With the help of the slicing skill, I upload it as an Issue in the format the loop understands, with the slices already cut. This is where I spend the most time, by far.

Investing here saves me work later. Every doubt I resolve before implementing is one less correction or redirection once the code is already written. And if the cut is bad, I pay for it several PRs later.

I decide the slicing, its order and its dependencies, and I keep working in **small increments**. Each slice carries its value, its scope, its acceptance criteria and an **observable verification**: which checks prove it works once merged. Keeping them small is what keeps the review manageable and lets every merge go to production safely.

### 2. The merge is my decision

Every slice ends in a pull request. Before I see it, another agent has reviewed it and left its verdict. But the decision is mine: merging approves, closing rejects, and a comment with a directive asks for another round.

Auto-merge exists, but it is off by default, and that is deliberate. I only turn it on when it pays off: repos without automatic deployment, with branch protection, and where undoing is cheap.

### 3. Every correction becomes a convention

This is the part that has changed my day-to-day the most. When I correct an agent in a review, I can mark that correction as a lesson. An agent turns it into a pull request against the repository conventions, which I review like any other.

The idea is simple: I don't want to correct the same thing twice. It is continuous improvement applied to the way of working itself, and it connects with something I was already doing: documenting conventions in plain files, with imperative rules and a single positive example per rule.

### 4. Determinism where possible

"Deterministic loop" does not mean the agents are deterministic. It means the orchestrator is boring code: on every tick it reads GitHub, detects what has changed and acts. Every step is idempotent. GitHub is the source of truth, and there is no broker, database or webhooks.

The non-deterministic part stays locked inside each slice, in an isolated worktree, with a limited number of retries and correction rounds. What happens between slices, in what order and when, does not depend on an agent remembering.

## What about TDD?

It is still there, but I no longer drive it test by test. The discipline I used to enforce in every cycle is now written down: in the persona of the implementing agent, in the TDD and verification skills it loads before starting, and in the repository conventions. The reviewer agent and I check afterwards that it has been followed.

I have traded fine-grained control for control at the points that matter most. What I used to watch at every step, I now watch in the spec and in the review.

The sense of ownership I defended in that post has changed, and I don't want to sugarcoat it. It is unavoidable: the further you move from writing the code, the less control you have over it. I no longer know every corner. What makes up for it is the brainstorming, which forces me to get into the change enough to know how the pieces fit. And the conventions give me confidence that they are built the way they should be.

## Why "groove"

I am a percussionist. In a band, percussion holds the pulse from within and everyone leans on it. That is the groove: playing without friction. That is what I wanted for delivery: a change that flows from the spec to the merge without stalling between one step and the next.

## Final thoughts

What this loop has given me is not that code gets written faster. It is that nothing sits waiting for me to remember, and that my time goes to what is still mine: thinking the change through, cutting it well, deciding what goes in and teaching the system not to repeat mistakes.

**I decide. The agent works for me, by my rules.**

If you are interested, there are more details on the [project website](https://grooveloop.pasmontesinos.com), and the code is on [GitHub](https://github.com/pasmontesinos/grooveloop) and [PyPI](https://pypi.org/project/grooveloop/). And if you work in a similar way, or a very different one, I would love to hear where you put your focus.
