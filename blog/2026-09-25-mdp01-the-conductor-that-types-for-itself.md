---
layout: post
title: "The Conductor That Types For Itself"
date: 2026-09-25
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [evolution, soredium, agent-workflow, devtown, conductor]
series: issue-1141-cdi-event-wiring
---

# The Conductor That Types For Itself

The generalisation work is done. Five SPI interfaces, five registries, the backbone rewired for domain-agnostic operation, follow-up migrations complete. Code-evolution is now a plugin, not a hardcoded assumption. The interesting question isn't what we built — it's what it implies about how agents could use it.

The evolution conductor proposes improvements based on health signals: a sensor degrades, proposals surface, gates filter, execution happens, regression detection watches the result. The orchestration backbone is domain-agnostic. What it lacks is an executor that knows how to implement an improvement safely — brainstorm before building, test before implementing, review before merging.

That methodology already exists. I've been building it for months under a different name: soredium. Skills that enforce TDD. Protocols that capture project conventions. A lifecycle that gates every branch from start to close. A garden that retains what worked and what didn't across sessions. The discipline a developer session follows — brainstorm, plan, TDD, review, merge — is exactly the discipline an autonomous improvement agent needs.

The connection is mechanical. The conductor determines *what* to improve and *when*. Soredium determines *how*. An agent executing an improvement proposal is a Claude Code session — same skills, same hooks, same safety envelope. The conductor types `work start #1234` into a terminal and the methodology takes over. No parallel API. No special agent framework. The executing agent gets TDD, code review, evidence-before-claims, and the full garden of prior knowledge for free, because it's just a session.

The hard part isn't the mechanism — it's the handoff. One LLM needs to type text in the terminal of another. The conductor observes health, decides an improvement is worth attempting, and initiates a session. The session runs under full soredium discipline. The conductor watches the result. If regression is detected after merge, it types `work start` on the revert issue in another terminal. Same loop, different direction.

Where this gets tested first matters. Devtown is the natural proving ground — a developer tool that uses the evolution conductor to observe its own development pipeline. Design specs land, code changes follow, PRs get built, health sensors watch the outcome. The components we build to observe the dev pipeline are the same components a trading desk would use to observe strategy performance. Blocks-ui gets the composable primitives; devtown composes them into the developer-facing workbench.

I don't have high expectations. Current models may not reliably execute multi-step improvement proposals end to end. But the infrastructure — conductor, SPIs, soredium discipline — is scaffolding that makes the experiment repeatable. Every failed attempt is a garden entry, a CBR case, a calibration signal. The methodology captures *why* it failed, not just that it failed. When a more capable model arrives, accumulated knowledge carries forward. Low expectations, high observability, cheap reversal.
