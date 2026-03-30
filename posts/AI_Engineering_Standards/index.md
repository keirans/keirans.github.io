---
layout: post
title: "The Cheapest AI Upgrade Is Already in Your Backlog"
date: 2026-03-30
categories: [AI, Engineering, DevOps]
tags: [AI Agents, ADR, Standards, Skills, MCP, CI/CD, Agentic Development]
author: Keiran
---

## Introduction

There is a common observation I am seeing in many of the AI tooling conversations happening right now: that the value of these tools is roughly equal across teams. That you point an agent at a codebase, give it a task, and the quality of the output is mainly a function of the backend model or the tool you chose.

From what I have experienced and seen in my day-to-day work, this perception is quite far from reality.

In my observations, the teams getting the most out of AI-assisted and agentic development today are not necessarily the ones with the biggest budgets or the most sophisticated prompt engineering. They are the ones who, years before any of this tooling existed, did the unglamorous work of defining how they build and deploy software, and wrote it down.

I often hear from other teams about patchy, inconsistent agent output, and the instinct is to treat it as a model or tooling problem: "Let's try it with Copilot instead, maybe rerun it on Kiro - it could be better..." From working with the ever-evolving coding agents over the last 18+ months, that has not been my experience. When we have seen inconsistent output when working on our activities, we generally can trace it directly to a gap or ambiguity in our written standards - not to the agent itself. The coding agents' maturity for many tasks now is quite consistent. Our standards initially were not.

This post is about that work. About why your engineering foundations -  your standards, your ADRs, your pipelines, your documentation -  are not just good hygiene anymore. I believe they are the single highest-leverage investment you can make in an AI-augmented engineering organisation.

If you cannot explain your ways of working clearly enough for a new human engineer to follow them independently, an AI agent unfortunately will not save you. It will amplify the challenges you probably already have, and likely at great cost to boot.

---

## Standards and Style Guides Are Agent Instructions

Every engineering team that has been operating for a period of time has opinions about how code should be written, how infrastructure should be laid out, and how access should be governed -  variable naming, error handling patterns, how to structure a module, when to use one library over another, how to write a commit message.

In most teams, these opinions live in a handful of places: a style guide document that exists somewhere, a set of linter rules, the comments in code review, and the institutional memory of the engineers who have been around the longest.

When AI coding assistants arrived, the better tooling vendors gave teams a way to externalise those opinions into a rules file -  a `CLAUDE.md`, a `.cursorrules`, a `copilot-instructions.md`. The framing was new. The concept was not. This was always just a style guide. The difference is that now the consumer of that style guide is not only a human onboarding to the team -  it is every agent invocation, on every task, every time.

The evolution of how we handled this in our team followed a predictable arc: human-readable documentation that engineers were expected to read and follow → linter and formatter configs that automated enforcement of the easily-automatable rules → coding assistant rules files that translated the harder-to-automate opinions into explicit agent instructions → centralised rules formats shared across tools and teams → agent Skills that made those standards dynamically loadable and actionable rather than statically injected.

Each transition was forced by the same realisation: a standard only has value if it is enforced at the point of work. As the point of work moved from human keystrokes to agent invocations, the enforcement mechanism had to move with it. This applies equally to your broader architectural standards - which brings us once again back to ADRs.

---

## If It Is Not in an ADR, It Does Not Exist for an Agent

Architecture Decision Records (ADRs) are the mechanism by which engineering teams capture not just what they decided, but why they decided it, and what alternatives they considered and rejected.

In an agentic engineering environment, ADRs serve the same function as they do for a new human engineer -  but the new engineer is every agent, on every task. And unlike a human who might seek out an ADR when they notice something unusual or gets pull request feedback on something they are working on, an agent will not go looking for context it does not know it is missing. It needs the relevant ADRs present in its context, or encoded into its Skills.

The distillation activities are important here. A full ADR with historical context and alternatives considered is valuable for humans. For an agent, the relevant extract is often two sentences: the decision, and the constraint it imposes. But done well, it can go further than that - a well-distilled Skill does not just tell the agent what the decision was, it gives it enough context to make the right choice independently, or better still, the exact steps to follow to arrive at the correct outcome. Loading the full ADR into the context window of every invocation is expensive and counterproductive - you are paying tokens to re-teach the model things it does not need for this specific task. Skills solve this by surfacing precisely the right guidance only when the task domain requires it.

But here is the prerequisite that is easy to miss: this only works if the ADRs exist in the first place. You cannot distil what was never written down. You cannot encode into a Skill a decision that only ever lived in an engineer's head.

If your team cannot hand a new human engineer a set of documents that explain how you build and deploy software and why - you are not ready to hand those same explanations to an agent.

---

## Automation Is What Standards Enable

Automation-first as a principle predates AI tooling by decades. The argument has always been consistency, repeatability, and reduced toil. Those arguments remain valid. But there is now a fourth and more immediate reason: AI agents can only do what is automatable.

Teams that committed early to automating their infrastructure provisioning, their deployment pipelines, their secret management, and their environment configuration find that agents slot into those workflows naturally, or at least with less friction. The automation is already the interface. The agent just calls it.

Teams that did not must now choose: invest in the foundational automation work before they can unlock agentic value, or accept that agents will only ever operate in a narrow, low-impact slice of their development lifecycle - or worse, they spend time extending the agent's capabilities to bridge manual steps with ugly workarounds like browser automation to click through forms and wizards.

Another important note to consider is that your CI/CD pipeline in particular becomes a key safety net. AI agents, and the LLMs that back them, are broadly fluent - they produce well-structured, syntactically correct, plausible code.

The errors they make are contextual, not syntactic: code that is architecturally wrong, tests that pass but do not test the right thing, configurations that work in isolation but conflict with something elsewhere in the system. A strong pipeline with real quality gates catches these. Teams with mature pipelines find that agents fit into their development loop - the pipeline provides feedback, the agent responds to it, and the loop converges on quality.

It is my observation that teams without mature pipelines find that agents produce volume, not quality.

---

## Your Standards Are an Asset You Can Hand to a 3rd Party

There is another dimension to this that many teams do not think about until they are already deep into a vendor or partner engagement: if you make your standards available in a portable human and AI format, the right vendors should be able to consume them directly.

When you engage a third party - a systems integrator, a managed service provider, a specialist consultancy - you are handing them a context problem. They need to understand how you build software, deploy cloud resources, the constraints you operate under, your ways of working, and what good looks like in your environment. Traditionally this has meant lengthy onboarding sessions, thick standards documents, and a slow period of back-and-forth before the quality of their output meets your bar.

AI-friendly standards and Skills change this. A vendor who receives your `CLAUDE.md`, your rules files, and your relevant Skills can onboard their agents - and their human engineers - to your standards immediately. The output arrives already aligned to your architecture, your conventions, and your constraints, rather than requiring rounds of review feedback to get there.

This also changes what you should be asking of vendors during procurement. A vendor who has invested in AI-assisted delivery practices should be able to demonstrate that they can consume your standards in this way. A vendor who cannot is possibly carrying hidden review and rework costs that could land on your team.

---

## How This Works in Practice (aka Dogfooding)

The approach described above is not theoretical for my current team - it is how we work day to day.

All of our development and deployment standards are defined as ADRs in a common Git repository, following a well-defined process for authoring, approving, and sunsetting them. This is the source of truth. Everything flows from here.

Inside that same repository, we maintain a set of generalised AI agent rules and Skills derived directly from those ADRs. We automate the generation of these using AI tooling, submitted through the same PR process as any other contribution. Critically, every rule and every Skill can be traced back to its source ADR - which means when agent output is inconsistent or wrong, we have a clear audit trail to diagnose whether the standard is missing, ambiguous, or simply not yet encoded.

For task execution, rather than engineers running agents on their laptops, we use an agentic task platform integrated with GitHub Projects. When a well-scoped ticket is ready, it can be assigned directly to an agent by moving it on the board. The platform handles provisioning of a task-specific runtime in a Kubernetes cluster - a Linux environment containing the tools, access, MCP configuration, and AI agent needed to perform that task. As part of that provisioning, the ADR repository is cloned and the full set of current rules, Skills, and knowledge is injected into the runtime automatically.

This architecture gives us two properties we care about deeply. The first is decoupling - agents run on dedicated infrastructure rather than engineer workstations, which eliminates device resource contention and allows multiple tasks to run in parallel without impacting anyone's local environment. The second is consistency - every task that runs gets the latest approved rules and Skills from the source of truth in git at launch time. There is no "sorry, I forgot to update my local config" failure mode. The standards are not something engineers have to remember to maintain on their devices. They are a core part of the infrastructure and software codebase.

The result is that the quality and consistency of agent output is a function of the quality of the standards in the repository - which is exactly where that responsibility should sit.

This architecture also positions us well for the convergence happening across the AI agent ecosystem right now. MCP, Skills, common ruleset formats, and spec-based development kits are rapidly becoming the standard interfaces through which agents consume context, access tooling, and understand how to behave. The major agent platforms - Claude Code, Kiro, GitHub Copilot, and others - are aligning to these standards rather than maintaining bespoke integration models. For teams that have invested in their foundations, this convergence is a tailwind: the same ADR-derived rules and Skills we maintain in our repository can be consumed by any agent platform that supports these standards, with minimal rework. For teams that have not, each new agent platform is another onboarding problem to solve from scratch.

Centralising your standards in a well-maintained repository and expressing them in open, portable formats is not just good practice for today's tooling. I feel that it is the right bet on where the ecosystem is heading.

---

## Your Engineering Foundations Matter More Than Ever

There is a framing that recurs in conversations about AI tooling: teams want to know which model to use, which agent tooling to adopt, how to structure their prompts. These are real questions with real answers. But I do feel that they are second-priority questions.

The first priority question should be: "Are your engineering foundations in place?"

The cost of establishing good foundations is a one-time investment that compounds. Every agent invocation that benefits from those foundations gets the benefit at zero marginal cost. The cost of operating without them is paid on every invocation - in review time, in rework, in incidents caused by agents confidently implementing the wrong thing.

Spec-driven development - the model where an agent implements against a clear specification - is not a distant future. It is available today, to teams whose foundations are in place. For those teams, the agent is a capable implementer who follows your standards, respects your decisions, and works within your architecture. For teams without those foundations, the agent is an enthusiastic junior who produces a lot of output and requires a lot of supervision.

I hand on heart feel that the cheapest, highest-impact thing you can do with AI today is not to find a better model or a better agent framework. It is to get your house in order.

If you are recognising gaps, start where the pain is most acute. Write the ADRs for decisions already made - the value of capturing a decision does not diminish because it was made in the past. Audit the agent output you have been reviewing and identify patterns of error - each pattern is a missing rule. Add the pipeline quality gates you know you should have.

The teams that will benefit most from AI are not the ones who adopt it fastest or spend the most on the best models. They are the ones who did the work - before any of this tooling existed - to define, document, and enforce how they build and release software.

If you can explain your ways of working clearly enough for a new engineer to follow them independently on day one, you can explain them to an agent. If you cannot, start there. The foundations matter more now than they ever did.

---

This post was written with assistance from AI, and I’ve worked to made sure all examples, configurations, and recommendations are technically accurate as of the time of writing.