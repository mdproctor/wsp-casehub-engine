---
layout: post
title: "Tenancy Threading Gets Explicit"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [tenancy, multi-tenancy, spi, cdi, quartz, blackboard]
---

My first instinct for tenancy in the engine repositories was elegant: inject
`CurrentPrincipal`, read `tenancyId()` in each query, filter silently. Callers stay
clean. Protocol PP-20260520-e6a5f0 says exactly this — bind tenancyId inside the
data access layer.

Going through the design with Claude in the spec review caught that it wouldn't
survive the codebase. The engine's repositories are called from `@ObservesAsync`
CDI observers, Quartz job listeners, and startup recovery services. None of those
contexts have an active CDI request scope. `currentPrincipal.tenancyId()` on any
of those threads throws `ContextNotActiveException` at runtime, not at compile time.

We switched to explicit: `String tenancyId` on every SPI method, callers supply it.
The HTTP boundary reads `currentPrincipal.tenancyId()` once; domain objects carry it
thereafter. Async observers get it from `event.tenancyId()`. Quartz jobs store it in
the job data map at scheduling time and read it in listeners via
`context.getMergedJobDataMap()`. More verbose, works in every context.

## The Cascade

`PlanItemRecord`, `PlanItemSaveRequest`, `PlanExecutionContext`, and `CaseLifecycleEvent`
are Java records. Adding a component breaks every construction site — forty-plus test
files across six modules. Most was mechanical, but a Python script for fixing call
sites hit a subtle edge: `[^)]+` stops at the first `)` regardless of nesting depth.
`findByUuid(instance.getUuid())` became `findByUuid(instance.getUuid(, tenancyId))`.
Claude caught some of these during the review cycle; the lasting fix was replacing the
regex with explicit `str.replace()` for each known pattern.

The cross-tenant SPI interfaces (`CrossTenantEventLogRepository`,
`CrossTenantCaseInstanceRepository`) were initially placed in
`runtime/internal/recovery/spi/` to restrict access. We had to move them to
`common/spi/` — `persistence-hibernate` needs to implement them, and the dependency
direction runs the other way.

## The Invariant That Silently Breaks Completion

Subcases inherit tenancyId from the parent case. Not from `currentPrincipal` — from
`parentInstance.tenancyId`. When subcase completion fires, `SubCaseCompletionService`
calls `findByWorkerAndType(childCaseId.toString(), SUBCASE_STARTED, event.tenancyId())`.
That event log entry was written under the parent's tenancyId. If the child was saved
with a different tenancyId, the lookup returns empty, the parent stays WAITING, and
the case never progresses — no exception, no prominent log entry.

I captured this as a protocol: `SubCaseExecutionHandler` must propagate
`parentInstance.tenancyId` to the child, never read from `currentPrincipal`. It's the
kind of constraint that passes code review because the incorrect path still compiles.

## A Spec Review Catch Before the Code Was Written

The original `BlackboardRegistry` design used `currentPrincipal` in `evict()` to
compose a `tenancyId:caseId` map key. A second review round caught the leak: if
eviction fires from a terminal-state handler without CDI request scope, the composed
key doesn't match the stored key and the entry is never removed — the registry grows
without bound.

The fix: store tenancyId in `CaseEntry` at `getOrCreate` time, keep O(1) UUID-keyed
eviction (UUID uniqueness makes cross-tenant collision impossible), use the stored
value in `get()` for defense in depth.
