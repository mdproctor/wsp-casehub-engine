---
title: "The Guard That Did Too Much"
date: 2026-05-26
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [casehub, casehub-engine, signal-bridge, LoopControl, WAITING, blackboard]
---

The signal gaps in the engine — Qhorus human messages that never reach cases,
WorkItem escalations that freeze WAITING cases, M-of-N group outcomes that vanish
— all traced to eleven lines in `CaseContextChangedEventHandler`:

```java
if (!caseInstance.getState().equals(CaseStatus.RUNNING)) {
    return Uni.createFrom().voidItem();
}
```

The guard does two things. It prevents duplicate dispatches of already-in-flight
bindings — necessary in the pure choreography path where there's no PlanItem
tracking. And it blocks all evaluation for non-RUNNING cases. Those are unrelated
concerns, and satisfying the second one is wrong. A WAITING case with an active
blackboard has PlanItem tracking that already prevents re-dispatch. The guard was
carrying load it didn't need to carry.

The fix is to move the dedup into `LoopControl`, which already owns dispatch
decisions. `ChoreographyLoopControl` keeps the RUNNING-only restriction — no
PlanItem tracking, no dedup. `PlanningStrategyLoopControl` accepts RUNNING and
WAITING both, filtering bindings whose PlanItems are already in flight before
dispatching.

The interesting part was what "in flight" means. I expected it to cover RUNNING,
DELEGATED, COMPLETED, FAULTED, and CANCELLED — any state beyond PENDING. The TDD
cycle caught me: the first test asserting a COMPLETED PlanItem wouldn't re-dispatch
passed before I'd written any production code. That means the test is wrong.

`CasePlanModel.addPlanItemIfAbsent()` makes the contract clear: COMPLETED items get
replaced by a new PENDING one. A binding whose conditions hold again after completion
should fire again — iterative cases depend on it. The filter blocks RUNNING and
DELEGATED only. There's a pre-existing timing race this doesn't address: a second
CONTEXT_CHANGED arriving before a handler marks a PlanItem DELEGATED will still find
it PENDING and re-dispatch. That's engine#364 — proper fix requires a DISPATCHING
transient state, separate work.

The adapter fixes were simpler. `WorkItemLifecycleAdapter` was mapping ESCALATED
to `markFaulted()`. ESCALATED returns the WorkItem to PENDING with new candidate
groups — the task isn't done, it's reassigned. We removed it from the terminal
filter. We also added a `@ObservesAsync WorkItemGroupLifecycleEvent` observer for
M-of-N SpawnGroup outcomes: COMPLETED maps to `markCompleted()`, REJECTED to
`markFaulted()`. The adapter now handles the full lifecycle.

The Qhorus bridge is twenty lines: a CDI `@ObservesAsync MessageReceivedEvent`
bean that calls `CaseHubRuntime.signal()` for commitment-resolving types —
RESPONSE, DONE, DECLINE, FAILURE — on channels named `case-{caseId}/{purpose}`.
The signal writes to `context["channelMessage"]`; case definitions react via
`contextChange(".channelMessage")`. COMMAND, QUERY, and STATUS are excluded;
EVENT has null content by protocol.

Claude caught something during code review: milestone and goal evaluation could
now run for SUSPENDED and terminal cases. The old guard blocked them; the new one
didn't address them. We added the `RUNNING || WAITING` check back at the handler
level for milestones and goals. Bindings get their dedup from LoopControl;
milestones and goals get a lifecycle check at the handler.

One open thread: `ClaudonyReactiveCaseChannelProvider` maintains its own
`CHANNEL_PREFIX = "case-"` string, separately from `CaseChannel.CASE_CHANNEL_PREFIX`.
The bridge parses using the engine constant; the provider writes using its own.
That's claudony#139.
