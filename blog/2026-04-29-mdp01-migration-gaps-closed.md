---
layout: post
title: "Migration Gaps Closed"
date: 2026-04-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [migration, subcase, dlq, idempotency, quarkus-ledger]
---

Three things closed today. The idempotency window landed, DLQ replay landed, and SubCaseBinding is sitting in PR #199 waiting for review. All three were migration gaps from the original casehub design — work that had been parked since the engine rewrite. Getting them out of the gap list felt significant.

The implementation itself was mostly mechanical. What wasn't mechanical was a cascade failure that ate most of the afternoon.

## The day quarkus-ledger broke everything

When `engine/pom.xml` added `quarkus-ledger` as a compile dependency — necessary for `LedgerTraceIdProvider` — the JPA entities from that jar appeared on the test classpath of every module that depends on `engine`. That's `casehub-blackboard`, `casehub-resilience`, and `casehub-work-adapter`. Each one started failing with `Datasource '<default>' is not configured` at test startup, because Hibernate saw the ledger entities and demanded a connection it couldn't find.

The error message pointed at datasource configuration. The root cause was a transitive dependency bundling JPA entities alongside domain code. Those are different things and they belong in different jars — we'd written the rule into `casehub-parent/docs/PLATFORM.md` but quarkus-ledger predated it. We created issues in [quarkus-ledger#73](https://github.com/casehubio/quarkus-ledger/issues/73) and [quarkus-qhorus#128](https://github.com/casehubio/quarkus-qhorus/issues/128) for the proper split, applied an H2 workaround to four test modules to unblock CI, and moved on.

The fix for each module was the same three-part pattern: add `quarkus-jdbc-h2` and `quarkus-ledger` as test-scope dependencies, configure an H2 datasource in `application.properties`, add a `NoOpLedgerEntryRepository` stub with `@Alternative @Priority(1)`. That last part is worth knowing: in Quarkus CDI, `@Alternative @Priority(N)` on a class auto-activates it without any `quarkus.arc.selected-alternatives` configuration. Most people reach for the config property and don't know the annotation is sufficient.

## SubCaseBinding

The SubCaseBinding implementation closed two items at once: casehubio/engine#195 (the binding itself) and casehubio/engine#76 (the old "future epic" stub from before the rewrite). The data model already existed — `SubCase` with namespace, name, version, and a completion strategy. We added `waitForCompletion`, `inputMapping`, and `outputMapping`, moved the class from `casehub-blackboard` into `api` so `Binding` could reference it without a circular dependency, and wired up the engine side.

The execution path is three hops. `CaseContextChangedEventHandler` detects a binding with a `subCase` field and publishes `SubCaseScheduleEvent` instead of `WorkerScheduleEvent`. `SubCaseExecutionHandler` (casehub-blackboard) catches that, spawns a child `CaseInstance` via `CaseHubRuntime.startCase()`, and — if `waitForCompletion=true` — transitions the parent to WAITING. `SubCaseCompletionListener` observes `CaseLifecycleEvent` for child terminal states, applies the outputMapping against the child's final context, and resumes the parent via `CaseResumptionService`.

That `CaseResumptionService` extraction was worth doing separately. The WAITING→RUNNING transition logic was duplicated between the Quartz worker path and the SubCase path — extracting it means both call the same code, and the EventLog entry type is passed as a parameter so the audit trail stays accurate.

Two code quality findings that mattered. The handler needed `blocking = true` on `@ConsumeEvent` because it calls `toCompletableFuture().join()` — without it, the Vert.x event loop blocks and the child's `CaseLifecycleEvent` can't fire. Silent deadlock. And `SubCaseCompletionListener` originally used `findByTypes(SUBCASE_STARTED)` doing a full table scan to find the parent case — we replaced it with a new `findByWorkerAndType` SPI method that scopes the lookup to the child case ID stored as `workerId` in the started entry.

The significant thing about SubCaseBinding isn't the mechanics — it's that a `Binding` can now route to either a worker or a child case, and the engine handles both paths without the caller knowing which it gets. The distinction between orchestration (explicit work submission) and choreography (binding-driven) already existed. SubCaseBinding extends choreography into hierarchical composition: a condition fires, a child case starts, the parent waits, the child result flows back. That's a meaningful unit of modularity for cases complex enough to warrant delegation.
