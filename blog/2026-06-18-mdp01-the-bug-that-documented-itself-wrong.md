---
layout: post
title: "The Bug That Documented Itself Wrong"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [failure-cascade, worker-outcome, testing]
---

The failure cascade landed in the previous session — `WorkerOutcome` sealed type, `OutcomePolicy`, structured `_outcomes` state, agent exclusion. Four follow-up issues remained open. All four turned out to be straightforward, but the session's real finds were two pre-existing bugs hiding in plain sight.

## The Follow-Ups

Recording success after a reroute (#514) was the headline issue but the simplest fix — when a worker succeeds on a binding that has prior failure data in `_outcomes`, update the status to COMPLETED and append the successful agent to the history. Pure observability; nothing breaks without it, but the dispatch audit trail now shows the full story.

The stuck PlanItem (#520) was more interesting. `PlanningStrategyLoopControl` marks PlanItems RUNNING *before* `publishWorkerSchedule()` filters candidates. If the exclusion filter removes everyone and `tryProvision()` is a no-op, the PlanItem is RUNNING with no Quartz job and no way to recover. The fix: detect exhaustion at dispatch time and publish `WORKER_OUTCOME_RESOLVED(EXHAUSTED)` directly, so the blackboard can fault the PlanItem. This closes the gap for both the new failure cascade path and a pre-existing scenario where all workers are unavailable during dispatch.

Stage reset cleanup (#517) was a handler in the blackboard — `StageResetOutcomesCleaner` consumes `STAGE_ACTIVATED` and removes `_outcomes` entries for the stage's bindings when `instanceIndex > 0`. Without it, agents excluded in iteration N stay excluded in iteration N+1. The fourth issue (#522) was already fixed — the tests pass on main.

## The Bugs That Were Already There

The real surprise was what the full test suite revealed after the follow-ups were done.

`GoalReachedEventHandler` was publishing `CaseStatus.COMPLETED` for failure goals. One enum value — `COMPLETED` instead of `FAULTED`. The test (`CaseFaultedStateTest`) was written correctly and had been failing since the feature was introduced, but the CLAUDE.md documentation described the buggy behaviour as the intended design: "Failure goals produce COMPLETED, not FAULTED — FAULTED is reserved for infrastructure faults." The doc had reverse-engineered a rationale for the bug.

The second one was a Vert.x codec registration crash that silently killed every blackboard integration test. `BlackboardEventCodecRegistrar` calls `registerDefaultCodec()` in `@Observes StartupEvent`, which works on first boot — but in `@QuarkusTest`, the Vert.x event bus instance is shared across application restarts. Second boot throws `IllegalStateException: Already a default codec registered`. The symptom chain is misleading: the first test class runs fine, all subsequent classes fail to start, their tests report as errors (not failures), and dependent tests are skipped. Thirty tests silent. The fix is a try-catch.

That codec gotcha was non-obvious enough to go into the garden — the connection between `@QuarkusTest` lifecycle, Vert.x instance reuse, and codec registration isn't documented anywhere in the Quarkus or Vert.x docs.
