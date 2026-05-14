---
layout: post
title: "Sub-case coordination — the race condition sequential tests miss"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [java, quarkus, concurrency, sub-case]
---

The session started with a false premise. The handover flagged SubCaseExecutionHandler as "never implemented — scaffold only." It was actually done: 142 tests green, parent cases transitioning to WAITING, children spawning correctly. What was missing was the parallel case — a parent spawning N children and waiting for M of them to complete.

That's what clinical needed for multi-site orchestration.

**The design question that mattered**

The first real question was whether to use quarkus-flow for the coordination logic. I didn't want to rebuild what a workflow engine already does well. But the coordination here is just counting: completedCount ≥ requiredCount. That's addition. The work module already made the same call — its MultiInstanceGroupPolicy is a counter with a policyTriggered flag, no workflow engine required.

Where quarkus-flow earns its place is conditional sequencing: "if site A fails, spawn a fallback instead." That's branching and dependency — a counter can't express it. So the rule became: native M-of-N for simple thresholds, quarkus-flow for anything that needs a decision tree. The implementation sits behind a repository SPI so the choice can change later without touching callers.

**The concurrency problem tests can't show**

We built the coordination layer — SubCaseGroup entity, SubCaseGroupRepository SPI, memory and JPA implementations, the grouped path in SubCaseExecutionHandler and SubCaseCompletionListener. 157 tests green.

Then a fresh Claude did a code review with no session context. It came back with a critical finding: evaluate and markPolicyTriggered weren't atomic.

The flow in SubCaseCompletionListener was: increment counter → evaluate threshold → mark policyTriggered → resume parent. Two concurrent child completions could both increment, both see COMPLETED before either sets the flag, and both resume the parent. The tests didn't catch it because they called `onCaseLifecycle()` sequentially. In production, multi-site completions arrive in parallel — that's the entire point.

The fix was changing `markPolicyTriggered` to return `Uni<Boolean>` — whether this particular call actually set the flag:

```java
// Memory: synchronized CAS
synchronized (g) {
    if (g.isPolicyTriggered()) return Uni.createFrom().item(false);
    g.setPolicyTriggered(true);
    return Uni.createFrom().item(true);
}

// JPA: conditional UPDATE
SubCaseGroupEntity.update(
    "policyTriggered = true WHERE parentCaseId = ?1 AND groupId = ?2 AND policyTriggered = false",
    parentCaseId, groupId)
  .map(count -> count > 0);
```

The caller returns immediately if it gets false. No double-resume. PostgreSQL handles the conditional UPDATE atomically at READ COMMITTED isolation.

**One more quiet failure**

We hit something else worth naming: `@ObservesAsync` CDI events don't fire in `@QuarkusTest`. Not intermittently — never. The event publishes, the CompletionStage returns successfully, and the observer is never called. No error. The test just silently skips the listener logic you thought you were exercising.

The current workaround is injecting the listener bean and calling the method directly. It works, but it's pointing at a real design issue: the listener mixes CDI delivery plumbing with all the coordination logic. The right fix is extracting the logic into a constructor-injected service — the listener becomes a five-line delegator, the service is testable as a plain Java object, and the workaround disappears. That's the next thing to clean up.

Clinical's multi-site orchestration is unblocked.
