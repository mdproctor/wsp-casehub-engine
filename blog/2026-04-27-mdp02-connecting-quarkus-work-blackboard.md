---
layout: post
title: "Connecting quarkus-work to the blackboard"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [quarkus, cdi, quarkus-work, blackboard, testing]
---

The two upstream pieces Claudony needs from casehub-engine are now shipped: `CaseLifecycleEvent` for worker execution transitions, and `casehub-testing` to let downstream `@QuarkusTest` classes run the engine in memory without Docker or PostgreSQL.

The lineage gap was simple to diagnose. `CaseLedgerEventCapture` observes `CaseLifecycleEvent` and writes ledger entries — but the engine only fired those events for case-level transitions. Worker execution never fired them. Claudony's `JpaCaseLineageQuery` was querying for ledger entries that were never written. The fix was two call sites: `WorkerExecutionJobListener.jobToBeExecuted()` for the start event, `WorkflowExecutionCompletedHandler` for completion. Both fire `CaseLifecycleEvent` with `actorId = workerId` and `actorRole = "WORKER"`.

`casehub-testing` packages the in-memory repository implementations with `@Alternative @Priority(1)` — the intent being that downstream consumers add one dependency and get working repos without `quarkus.arc.selected-alternatives` config. Claude and I hit a wall immediately: the jar has no Jandex index, so Quarkus CDI can't see the beans. The fallback was the pattern every other module already uses — `casehub-persistence-memory` as a test dependency, activated via config. That works. The Jandex fix is deferred.

The more substantial piece was `casehub-work-adapter`. quarkus-work fires `WorkItemLifecycleEvent` CDI events when a WorkItem reaches a terminal state. CaseHub needs to receive those and translate them into `PlanItem` transitions in the blackboard. I had to choose between choreography and orchestration — write the child outcome to `CaseContext` and let the engine re-evaluate naturally, or register a `CompletableFuture` in `PendingWorkRegistry` and resolve it on the event. I went with choreography. The blackboard already works that way, and nothing needed the orchestration path yet.

The `callerRef` field on `WorkItem` is the routing key — CaseHub embeds `case:{caseId}/pi:{planItemId}` at spawn time, quarkus-work echoes it back unchanged. Parsing it routes to the right `CasePlanModel` via `BlackboardRegistry`. Status mapping: COMPLETED → COMPLETED, CANCELLED → CANCELLED, REJECTED/EXPIRED/ESCALATED → FAULTED.

Getting tests to run turned up two quarkus-work surprises. The issue description says to access `workItem().callerRef` on the event. Claude wrote the initial observer; the compiler rejected `event.workItem()` immediately. `WorkItemLifecycleEvent` stores `WorkItem` as a private field with no public accessor. We decompiled with `javap` and found that `source()` returns it — not in any documentation.

The second: `WorkItemStatus.EXPIRED.isTerminal()` returns false. EXPIRED is semantically terminal — miss it and you don't route expired work items to FAULTED. The bytecode tableswitch has EXPIRED in the false branch alongside DELEGATED and SUSPENDED. The adapter enumerates the statuses it cares about explicitly rather than delegating to `isTerminal()`.

The quarkus-work test setup added a third challenge: the full JAR brings JPA entities that need a datasource, and `JpaWorkloadProvider` clashes with the engine's `CasehubWorkloadProvider`. The fix was `quarkus-work-testing` for in-memory WorkItem stores, H2 as a test datasource, and an `@Alternative @Priority(1)` stub for `WorkloadProvider` as a static inner class in the test.

A null pointer in `CaseContextChangedEventHandler` surfaced during this — `ConcurrentHashMap.get(null)` when `instance.getCaseMetaModel()` is null. The existing null check on the returned definition came one line too late. Fixed.
