---
layout: post
title: "Phase 2: Resilience, Diff Provenance, and a Persistence Rethink"
date: 2026-04-15
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [architecture, casehub-engine, casehub-resilience, testing, persistence]
excerpt: "Three new modules designed and shipped — resilience, EventLog enrichment, and a persistence decoupling spec — plus a conversation that changed the ORM approach entirely."
---

I wanted to build on PR #49 without waiting for co-owner review. The persistence decoupling was next on the list, but that's next session. First, the work we shipped.

## casehub-resilience: backoff, dead letters, poison pills

The resilience module covers three failure patterns. Retry backoff (`BackoffStrategy.FIXED`, `.EXPONENTIAL`, `.EXPONENTIAL_WITH_JITTER`) plugs into the existing `RetryPolicy` record — the engine now computes delay correctly on reschedule rather than applying a flat wait. Dead-letter queue routes exhausted workers to an in-memory store with query, discard, and replay support; a `@ConsumeEvent` handler listens on `WORKER_RETRIES_EXHAUSTED` alongside the engine's existing fault handler. Poison pill detection quarantines workers that fail above a sliding-window threshold — the `WorkerExecutionGuard` SPI lets the resilience module block scheduling without the engine depending on it.

354 tests passing across the full stack. We split the work into three stacked PRs (#52, #53, #54) targeting `feat/rename-binding-casedefinition`, which required pushing that branch to upstream before GitHub would accept the stacked base.

One thing Claude caught during code review: the logger in `WorkflowExecutionCompletedHandler` was registered against `CaseStartedEventHandler.class`. Everything worked, but every error would have appeared under the wrong logger name. One-line fix; easy to miss indefinitely.

## Enriching the EventLog

`WORKER_EXECUTION_COMPLETED` previously carried the raw worker output and an idempotency hash. We added `contextChanges` — a before/after diff of every top-level key that changed. The `ContextDiffStrategy` SPI gives three options: `TopLevelContextDiffStrategy` (the default, per-key `{ before, after }` objects), `JsonPatchContextDiffStrategy` (RFC 6902 array via zjsonpatch, already on the classpath), and `NoOpContextDiffStrategy` for when the overhead isn't wanted.

Activating an alternative via `quarkus.arc.selected-alternatives` — same pattern as the PoisonPill guard. But a CDI 4.0 detail caught us: `@Alternative @Priority(n)` doesn't rank the bean, it *globally activates* it. We had both alternatives annotated this way, which made all three implementations simultaneously active and caused `AmbiguousResolutionException`. Removing `@Priority` from both alternatives fixed it. PR #56.

## The persistence conversation

Designing `casehub-persistence-memory` forced a real question: the engine's entities extend `PanacheEntity`, which means any module depending on engine also pulls in Hibernate and PostgreSQL. No clean in-memory path exists.

My first design used `orm.xml` to externally map the engine's domain objects to database tables — no JPA annotations on `CaseInstance`, `EventLog`, or `CaseMetaModel`, but Hibernate still maps them from the persistence module side.

Francisco Javier Tirado Sarti — the quarkus-flow co-creator — disagreed, and he was right. The `orm.xml` approach doesn't solve the detached entity problem. If `CaseInstance` is Hibernate-managed, accessing lazy relationships after the session closes throws `LazyInitializationException`. Since `CaseInstance` lives in `CaseInstanceCache` indefinitely, the session is always gone.

His approach, visible in the quarkus-flow source: separate entity classes in the persistence module (`CaseInstanceEntity` with JPA annotations, `@DynamicUpdate`, extending `PanacheEntity`), converters between entity and domain object, and pure POJOs in the engine core. The domain objects have zero JPA awareness. He used this pattern across quarkus-flow, the serverlessworkflow sdk-java, and DataIndex — three production systems — so I'm not second-guessing it.

The spec is written. Two new modules: `casehub-persistence-memory` (in-memory `ConcurrentHashMap` implementations, no Docker for tests) and `casehub-persistence-hibernate` (JPA entities, Flyway migrations moved from engine, the only place that needs PostgreSQL). Implementation plan next session.
