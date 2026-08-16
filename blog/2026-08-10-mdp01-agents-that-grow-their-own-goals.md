---
layout: post
title: "Agents That Grow Their Own Goals"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [goal-lifecycle, agent-autonomy, reflection, goal-formation]
series: issue-800-agent-learning-subepics-bc
---

The landscape analysis I did in July surveyed every multi-agent project worth examining — Smallville, Emergence World, Concordia, the lot. One gap stood out from the rest: no project has agents that form new goals from experience. Goals are pre-declared by the developer or implicit from survival pressure. Nobody closes the loop.

Today we closed it — or at least built the mechanism. Goal formation is the pipeline that turns accumulated experience into new objectives: reflection produces insights, the LLM extracts goal candidates from those insights alongside the agent's existing goals and recent memories, validation enforces the structural constraints (name length, description bounds, no duplicates, max 10 goals per agent), and registration writes the new goals onto the AgentDescriptor.

The interesting design question was where to draw the line between mechanism and trigger. The issue tracker had these as two separate items — one for the formation mechanism and one for goal discovery from memory (the trigger path from reflection). I merged them. The trigger and the mechanism are the same pipeline. Reflection insights ARE the formation context. Designing the SPI to accept generic context and then always passing it reflection insights would be indirection for its own sake.

The approval gate was the decision I spent the most time on. Three options: auto-register everything (fast but risky for a capability no one has battle-tested), route through ActionGate (the existing human approval workflow), or a config flag. I went with the config flag. ActionGate is the wrong abstraction — it's case-level, creating WorkItems on specific cases, but goal formation is agent-level. It spans across cases. A config property gives us `true` for development and `false` for production, where goals are logged as `GOAL_PROPOSED` events but never registered. External systems can observe and build their own approval workflow. No commitment to a UX we might get wrong.

The other guard rail worth noting is the rate limiting. A per-reflection cap (default 2 new goals per cycle) prevents a single reflection from flooding an agent with goals, and a configurable cooldown (default 1 hour) prevents sustained high-frequency formation. On top of that, the hard ceiling of 10 goals on `AgentDescriptor` is the final safety net — `toBuilder().goals(merged).build()` validates at construction time.

The architectural pattern is worth calling out because it's becoming the standard in this codebase: an `@ApplicationScoped` evaluator injected into the handler chain, called on a virtual thread after the upstream operation completes. `GoalFormationEvaluator` follows the exact shape of `GoalRevisionEvaluator` and `PersonalitySignalRecorder` — the evaluator-per-completion pattern. The evaluator manages its own cooldown state via `ConcurrentHashMap`, spawns its own virtual thread for the heavy work (LLM call, memory retrieval), and never blocks case progression. Three components now share this pattern, which suggests it's the right level of abstraction for agent-level learning triggered by case-level events.

The formation pipeline completes the goal lifecycle. Agents can now have their goals revised based on outcomes, abandoned when infeasible, and formed from experience. The lifecycle is open: goals emerge, adapt, and retire without developer intervention. What's still missing is the feedback loop — do formed goals actually improve agent performance over time? That's not something you can answer in a design spec. It needs agents running in production, accumulating real experience, forming real goals. The infrastructure is in place. The experiment starts when it ships.
