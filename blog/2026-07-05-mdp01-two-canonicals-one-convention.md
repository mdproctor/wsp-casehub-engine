---
title: "Two Canonicals, One Convention"
date: 2026-07-05
author: Mark Proctor
type: diary
projects: [casehub-engine]
tags: [persistence, spi, dual-stack, convention]
series: issue-651-cross-repo-blocker-batch
---

# Two Canonicals, One Convention

The naming convention is straightforward: unqualified name is blocking, `Reactive` prefix is `Uni`-based. `PlanItemStore` established this months ago. The interesting part is what "canonical" means when you have two persistence layers that work in opposite directions.

In-memory repositories are synchronous. The `ConcurrentHashMap` operations complete immediately. Wrapping them in `Uni.createFrom().item()` adds nothing — it's pure ceremony. So the blocking class owns the state, the locks, and the logic. The reactive class injects the blocking one and wraps every call.

JPA is the reverse. Hibernate Reactive Panache returns `Uni<>` natively. The reactive class *is* the implementation — it owns `withTenantTransaction()`, the entity mapping, the JPQL queries. The blocking class injects the reactive one and calls `.await().indefinitely()`.

Same convention, different canonical. The caller doesn't care which side owns the logic — the interfaces are identical minus the `Uni` wrapper. But if you need to understand *how* something is stored, you read the canonical: `InMemoryCaseInstanceRepository` for the in-memory path, `JpaReactiveCaseInstanceRepository` for the production path.

## The rename that renamed too much

The first step was renaming the existing interfaces from `CaseInstanceRepository` to `ReactiveCaseInstanceRepository` — freeing the unqualified name for the new blocking interface. IntelliJ's `ide_refactor_rename` with `relatedRenamingStrategy: "all"` propagated this across every import, injection site, and `selected-alternatives` entry across 26 repos in the workspace. Clean — until we checked the implementation class names.

`InMemoryCaseInstanceRepositoryReactive` became `InMemoryReactiveCaseInstanceRepositoryReactive`. The rename tool treats the interface name as a substring and replaces it everywhere it appears in a class name, including classes that already had a suffix convention of their own. Six classes got double-Reactive names. A second rename pass fixed them all, but it's the kind of thing that compiles cleanly and only surfaces when you actually read the class names.

## Cross-repo migration shapes

The consumer migration for `StaticSetStrategy.of()` split into two patterns. Repos like devtown and iot construct `RiskDecision.GateRequired` directly — `List.of("iot-operator")` becomes `StaticSetStrategy.of("iot-operator")`. Mechanical, one-to-one replacement.

The enum-based repos — aml, clinical, life, soc — delegate through an `ActionType` enum. The `List<String>` field type, constructor parameter, and accessor return type all change to `CandidateSetStrategy`. Each enum constant's `List.of(...)` becomes `StaticSetStrategy.of(...)`. Same semantic change, but the migration surface is the enum definition, not the classifier. The classifier itself doesn't change at all — it already calls `type.candidateGroups()` and passes the result through.

The dual-stack work unblocks `casehub-blocks` for AI routing strategy implementations, and the consumer migration means every strategy SPI in the platform now speaks the same `NamedStrategy` + `StrategyResolver` vocabulary. quarkmind is still on a feature branch — that migration is parked until it returns to main.
