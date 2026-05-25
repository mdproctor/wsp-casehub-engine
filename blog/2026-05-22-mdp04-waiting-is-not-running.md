---
layout: post
title: "Waiting Is Not Running"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [blackboard, state-machine, planitem, quarkus, cdi]
---

The previous branch left the `@QuarkusTest` suites in `casehub-blackboard`
blocked — `RoutingCursorStore` unsatisfied dependency. The fix was
straightforward: add `quarkus.arc.exclude-types=io.casehub.work.core.strategy.RoundRobinStrategy`
to the test configs. `RoundRobinStrategy` lives in `casehub-work-core` and
requires a `RoutingCursorStore` provider that doesn't exist in the engine's
test context. The proper fix is a `@DefaultBean` no-op in casehub-work, filed
as casehubio/work#214. For now, the exclusion lets CDI ignore it.

With the tests unblocked, we could see what the blackboard actually needed. The
stated problem was "SubCase PlanItems never transition beyond PENDING." Once we
looked, it was wider than a missing state transition.

Every SubCase and HumanTask PlanItem in the blackboard carried the status
RUNNING. That's wrong. RUNNING means a Quartz thread is actively computing. A
SubCase PlanItem is waiting for a child case to return — the engine has handed
off and gone idle. An LLM reading RUNNING on that PlanItem would infer active
local computation. It's a misrepresentation of what the system is doing.

I asked whether this needed a new state rather than a workaround. The answer
required a full systematic review — all target types, all terminal scenarios,
every error path. The result was a 6-state model: PENDING, RUNNING, DELEGATED,
COMPLETED, FAULTED, CANCELLED. RUNNING stays for CapabilityTarget only.
DELEGATED means "control passed to an external actor; the engine is waiting for
a completion signal." No ambiguity.

The implementation found a second problem immediately.
`DefaultCasePlanModel.getAgenda()` only returns PENDING items — the code
filtered on `status == PENDING`, full stop. When we tried to find a DELEGATED
PlanItem via the agenda stream, we got an empty result with no error. The fix
was to use `getPlanItemByBindingName()` instead, updating that method and
`hasActivePlanItem()` and `addPlanItemIfAbsent()` to treat DELEGATED as active
alongside PENDING and RUNNING. Without that, a DELEGATED PlanItem wouldn't
block the binding from spawning a second child case.

The completion path was also missing. Worker completion (CapabilityTarget)
already flowed through `PlanItemCompletionHandler`, which fires stage
autocomplete and a `PlanItemCompletedEvent`. SubCase completion didn't —
`SubCaseCompletionService` updated the parent case state but never touched the
PlanItem. We added a `SubCaseExecutionCompleted` event, published by
`SubCaseCompletionService` after the parent resumes, consumed by
`PlanItemCompletionHandler`. Both paths now use the same
`completePlanItemByKey()` method. Stage autocomplete fires for SubCase
bindings.

We also closed the error paths. If `startCase()` throws, or the CaseDefinition
doesn't exist, or the group configuration is invalid — the PlanItem was
previously left in PENDING. `hasActivePlanItem()` returns true for non-terminal
states, so a stuck PENDING binding never reschedules. Each error path now calls
`faultPlanItem()`.

One thing caught during the work: Java 21 pattern-matching switch doesn't match
`null` with `default`. A traditional switch treats `default` as the catch-all
for everything including null; pattern-matching switch doesn't — it throws NPE
unless `case null` is declared explicitly. The fix is `case null, default -> { ... }`.

The `HumanTaskScheduleHandler` was also calling `markRunning()` when creating
WorkItems. Fixed to `markDelegated()`.
