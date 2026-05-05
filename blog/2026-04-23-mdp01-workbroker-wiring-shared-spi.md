---
layout: post
title: "WorkBroker: wiring the shared SPI into casehub-engine"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [quarkus-work, WorkBroker, orchestration, choreography]
---

quarkus-work-api landed. We spent the session wiring it in.

The first thing to resolve was naming. casehub-core had `TaskBroker` — a concrete class that routed work to workers, handled idempotency, and checked poison pills. The shared SPI coming from treblereel's quarkus-work-core uses `WorkBroker`. The names aren't synonyms: `Work` is the generalised assignable unit, `WorkItem` is Work with human inbox semantics, and `Task` belongs at the sub-step level — inside a unit of Work. I filed ADR-0003 to close this and retire `TaskBroker` as a name. Future code uses `WorkBroker`.

That settled, we had two integration points: the choreography path and the orchestration path.

**Choreography** was the quick win. The engine's `CaseContextChangedEventHandler` was publishing a `WorkerScheduleEvent` for every capable worker when a binding fired — a schedule-all approach. `WorkBroker.apply()` replaces that: pass in a list of capable `WorkerCandidate`s, get back an `AssignmentDecision` with the one to schedule. `LeastLoadedStrategy` handles the selection, weighting by active Quartz job count per worker.

One thing not in the documentation: `WorkerCandidate.of(id)` creates a candidate with an empty capabilities set. If you pass the capability name as `requiredCapabilities` in `SelectionContext`, `WorkBroker` filters out every candidate — silently, no error, just returns `noChange()`. The fix is to pass `null` for `requiredCapabilities` when callers do their own pre-filtering. We also ran into CDI ambiguity: `quarkus-work-core` registers both `LeastLoadedStrategy` and `ClaimFirstStrategy` as `@ApplicationScoped` beans. Injecting `WorkerSelectionStrategy` (the interface) fails at startup. Inject the concrete type.

**Orchestration** was the architectural piece. casehub-core's `TaskBroker` was synchronous — `submitTask()` returned a `TaskHandle`, you called `awaitResult()`. casehub-engine is reactive and Quartz-based. The pattern doesn't port directly.

`WorkOrchestrator` is the replacement. `submit()` and `submitAndWait()` both use `WorkBroker` for selection, then publish a `WorkerScheduleEvent` through the existing Quartz scheduling pipeline. The result comes back as a `CompletionStage<WorkResult>` resolved by `PendingWorkRegistry` when the Quartz job completes.

The durability requirement meant the correlation had to survive a JVM restart. `PendingWorkRegistry` rebuilds from the EventLog on startup — it scans for `WORK_SUBMITTED` entries without a matching `WORK_COMPLETED` and re-registers futures for them. When `WorkerExecutionRecoveryService` replays the in-flight Quartz jobs, those futures resolve normally.

For case-internal orchestration, `submitAndWait()` also transitions the case to `WAITING` and persists a `waitingForWorkId` column on `CaseInstance`. When the worker finishes, `WorkflowExecutionCompletedHandler` checks whether the completing case was waiting for that specific correlation key. If so, it transitions `WAITING → RUNNING` before firing `CONTEXT_CHANGED` — the case resumes and bindings re-evaluate as normal.

The `quarkus-work-core` module also bundles a `FilterRule` JPA entity. Any module that transitively depends on it — including `casehub-resilience` via the `engine` module — triggers Hibernate ORM discovery and requires a configured datasource. Adding `quarkus.hibernate-orm.enabled=false` to the resilience test properties resolved it.
