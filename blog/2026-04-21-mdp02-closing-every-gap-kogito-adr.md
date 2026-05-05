---
layout: post
title: "Closing Every Gap: Parity, Kogito, and ADR-0002"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [blackboard, architecture, cmmn, kogito, adr]
---

The previous entry ended at 99 tests. I wasn't done.

A fair comparison of the two LoopControl implementations — ours and treblereel's
original — turned up four genuine gaps we hadn't addressed. I want to be precise
about what "genuine" means here: not design differences, not things we chose
differently, but capabilities the prior implementation had that ours simply
lacked.

**BlackboardPlanConfigurer** — treblereel's `CasePlanModelRegistry` let you
declare stages for a case definition at design time. We had no equivalent. PR-F
adds a CDI SPI: implement `@ApplicationScoped BlackboardPlanConfigurer`, override
`configure(CasePlanModel, PlanExecutionContext)`, and the loop control calls it
exactly once when a case instance starts.

**Strict PlanItem lifecycle** — treblereel threw `IllegalStateException` on
invalid transitions. Ours silently accepted anything via a raw `setStatus()`.
PR-G replaces the setter with `markRunning()`, `markCompleted()`, `markFaulted()`,
`markCancelled()` — each validates the current state and throws if wrong. The
implementation caught something: several tests were jumping directly from
PENDING to COMPLETED, bypassing RUNNING entirely. That's a test design flaw the
strict API correctly surfaces.

**SubCase** — a stage item representing a child case definition, with a completion
strategy mapping child case status back to stage item status. Pure data model —
the engine integration stays in the future epic. PR-H ports it for feature parity.

**Stage entry validation** — treblereel's Stage builder required an entry
condition. Ours silently used null, meaning "activates every cycle." PR-I adds
`Stage.builder(name)` which throws `IllegalStateException` if `build()` is called
without `entryCondition()`. `Stage.alwaysActivate(name)` replaces `Stage.create(name)`
for cases where always-activating is the intent — explicit, not silent.

Then the deeper comparison.

Reading treblereel's `PlanningStrategyLoopControl` carefully, I noticed it
split eligible bindings into staged (worker assigned to a stage) and free-floating
(worker not in any stage). Staged bindings were blocked when their stage wasn't
active. I'd assumed this was a consequence of the architecture we'd replaced.
I was wrong about the origin.

The Kogito flexible process blog posts explained the reasoning. This is the
CMMN ad-hoc process model: you declare all possible workflow fragments at
design time. Stages act as scope containers. When a stage activates, its
declared fragments become available. When it doesn't, they're invisible to the
selection loop. Free-floating fragments are always available — monitoring,
error handling, anything that should run regardless of stage state.

The decision was how to opt into this. We considered a flag on `CasePlanModel`,
a mode enum, and a convention. The convention wins: if a Stage has binding names
declared, it's a gating stage. If not, it's lifecycle-only. The presence of
`stage.addBinding("trigger-name")` IS the opt-in. Three modes from one
mechanism — pure choreography, fully gated, hybrid — with no flag and no enum.

ADR-0002 captures the reasoning. PR-J implements it: `Stage.addBinding()`,
`withBinding()`, `builder().binding()`, and gating logic in
`PlanningStrategyLoopControl.select()` — with a fast path when no stages have
declared any bindings (pure choreography, zero overhead).

120 tests became 136. Ten QE PRs open (#91–#100). The new implementation is
now strictly superior in every dimension the comparison identified.
