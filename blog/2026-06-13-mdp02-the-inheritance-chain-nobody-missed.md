---
layout: post
title: "The Inheritance Chain Nobody Missed"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Engine]
tags: [cdi, quarkus, refactoring]
---

The batch PR from earlier today shipped a composition refactor on `CaseLedgerEntryRepository` — breaking it free from `JpaLedgerEntryRepository` so tests could substitute an in-memory double without hitting native SQL tables. Clean separation. The kind of change that should only affect the module it touches.

CI disagreed. Eight unsatisfied `LedgerEntryRepository` injection points in `casehub-engine-flow` — a module that wasn't touched in the PR.

## The invisible coupling

`CaseLedgerEntryRepository` used to extend `JpaLedgerEntryRepository`, which implements `LedgerEntryRepository`. The class was `@DefaultBean @ApplicationScoped` — meaning it was discoverable by CDI and active by default. Every module on the test classpath that needed a `LedgerEntryRepository` got one for free through inheritance, whether or not it knew that's where the bean came from.

The composition refactor removed `extends JpaLedgerEntryRepository`. The class kept its own methods and its `@DefaultBean` annotation. But it stopped being a `LedgerEntryRepository`.

The next question was obvious: why didn't `JpaLedgerEntryRepository` itself — the actual implementation — satisfy the injection points? Claude ran `javap` on the published JAR:

```
public class JpaLedgerEntryRepository implements LedgerEntryRepository {
  ...
RuntimeVisibleAnnotations:
    jakarta.enterprise.context.ApplicationScoped
    jakarta.enterprise.inject.Alternative
```

`@Alternative` without `@Priority`. Never activated unless explicitly selected. The upstream library designed it as an opt-in implementation — the `casehub-engine-ledger` module was supposed to provide the active bean. Which it did, through inheritance. Until it didn't.

## The fix and the pattern

Four other modules — `runtime`, `blackboard`, `resilience`, `work-adapter` — already had `NoOpLedgerEntryRepository` stubs. They'd been added at various points when those modules hit the same classpath issue. The `flow` module never needed one because the inherited type chain had been silently satisfying the dependency.

The fix was mechanical: add the same `NoOpLedgerEntryRepository` stub to `flow/src/test/java/`, exclude the ledger capture beans per protocol PP-20260610-18a084. Twenty-four tests pass. CI green. PR merged.

The interesting part isn't the fix — it's the failure mode. `@DefaultBean` classes that implement an SPI via inheritance create invisible downstream coupling. The coupling doesn't show up in import statements, dependency declarations, or any static analysis. It shows up when someone refactors the class for perfectly good reasons and a module three hops away in the dependency graph fails at CDI deployment time.
