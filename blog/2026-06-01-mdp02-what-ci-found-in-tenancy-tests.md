---
layout: post
title: "What four CI failures found in the multi-tenancy test infrastructure"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [quarkus, cdi, flyway, multi-tenancy, testing]
---

The PR had seven commits and clean local tests. CI disagreed — four times.

The S/XS batch work itself was finished. Merging it was supposed to be a formality. What CI found instead was a sequence of test infrastructure gaps that adding `@Inject CurrentPrincipal` to the engine's core beans had quietly exposed. Claude and I worked through them one run at a time.

## A migration nobody wrote

`casehub-persistence-hibernate` uses a strategy the other modules don't: Flyway runs migrations first, then Hibernate validates the schema against entity definitions. It's deliberate — the combination catches entity/migration drift. The new `tenancy_id` columns on all five entity classes had no Flyway migration behind them.

We added V1.2.0: `ALTER TABLE case_instance ADD COLUMN IF NOT EXISTS tenancy_id VARCHAR(64) NOT NULL DEFAULT '__system__'`, and the same for `event_log`, `plan_item`, `subcase_group`, and `case_meta_model`. Two columns on `plan_item` had also slipped in without migrations — `target_type` and `output_mapping_expression`. Both nullable, never noticed because the other test modules use `drop-and-create` and recreate the schema from entity classes on every run.

## The CurrentPrincipal that only existed in one profile

The `runtime` module has two Maven test profiles: a default that runs against PostgreSQL via TestContainers, and a `persistence-memory` profile that runs in-memory with H2. `DefaultTestPrincipal` — the `@DefaultBean CurrentPrincipal` that returns the default tenant sentinel — lives in `casehub-persistence-memory`, which is only on the classpath in the memory profile.

The default profile had no `CurrentPrincipal` bean at all. Quarkus said "Unsatisfied dependency" and refused to start. The fix was making `casehub-persistence-memory` an always-on test dependency. Its SPI implementations are `@Alternative` — they only activate when listed in `selected-alternatives`, so they don't interfere with the Hibernate implementations in the default profile. Only `DefaultTestPrincipal`, which is plain `@DefaultBean`, activates automatically.

## Two @DefaultBean implementations, neither yielding

`casehub-platform` ships `MockCurrentPrincipal`, `MockGroupMembershipProvider`, and `MockPreferenceProvider` — all `@DefaultBean @ApplicationScoped`. They return null or uninitialised values; they're designed to be wired up by a test harness, not used standalone. In three modules — `blackboard`, `resilience`, `work-adapter` — these platform mocks were landing on the classpath alongside the engine's own `@DefaultBean` no-op implementations.

The `@DefaultBean` contract is: yield to any non-default bean. With two `@DefaultBean` beans and no non-default one, neither yields. Quarkus reports "Ambiguous dependencies" — not "Unsatisfied" — and refuses to start. I excluded all three platform mocks via `quarkus.arc.exclude-types` in each module's test `application.properties` rather than waiting to discover them one at a time.

While fixing `blackboard`, Claude caught something else: `SubCaseMofNIntegrationTest` and `SubCaseParallelIntegrationTest` were constructing `CaseLifecycleEvent` with `null` as the second argument. That position used to be `commandType`. This branch added `tenancyId` as the new second field, which shifted everything else down by one. Java records are positional — the compiler had no way to flag that a `null` intended for `commandType` now landed in `tenancyId`. The tests compiled; repository methods calling `tenancyId.equals(...)` did not.

The PR merged on the fifth CI run.
