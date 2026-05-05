---
layout: post
title: "Phase 2: Standards, a Hidden Bug, and casehub-blackboard"
date: 2026-04-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [architecture, casehub-engine, casehub-blackboard, testing]
excerpt: "Naming decisions resolved, CaseStatus aligned with CNCF standards, a silent bug found by the tests we wrote, and casehub-blackboard went from brainstorm to 390 tests in one session."
---

The session opened with a list of blockers — naming decisions that needed resolution before Phase 2 could start. Most turned out to be quick.

ConflictResolver: drop it. The async event cycle serialises all writes through the Vert.x event loop, so the race condition it was designed for doesn't exist in casehub-engine's model. ContextChangeTrigger: keep it. Once I'd decided the shared workspace should be called CaseContext rather than CaseFile or CaseState, the trigger name became naturally consistent — it fires when the context changes. Done.

## The standards case for RUNNING and FAULTED

CaseStatus needed evidence. The question was whether casehub-engine's values — ACTIVE, FAILED, TERMINATED — should align with casehub's — RUNNING, FAULTED, CANCELLED. I pointed Claude at the Quarkus Flow source at `~/dev/quarkus-flow`. It came back with a clean answer: Quarkus Flow imports `io.serverlessworkflow.impl.WorkflowStatus` directly from the CNCF reference implementation (v7.17.0.Final). The values: RUNNING, COMPLETED, FAULTED, SUSPENDED, CANCELLED.

casehub-engine's ACTIVE, FAILED, and TERMINATED were all non-conformant relative to a spec the project had implicitly committed to following. casehub's names were the right ones. We renamed the enum, wrote a Flyway V1.2.0 migration, and closed issue #14.

I also renamed `StateContext` to `CaseContext` — a name Java developers will recognise from `ApplicationContext` and `ServletContext` without documentation. The IntelliJ MCP index tools handled the semantic rename across 26 files cleanly.

## The bug hiding in the success

Writing tests for the FAULTED state transition found something. `WorkerRetriesExhaustedEventHandler` was calling `caseInstance.persist()` on a detached Panache entity — triggering an INSERT, hitting a duplicate-key constraint on the uuid column, and rolling back the transaction silently. The case showed FAULTED in the in-memory cache (the state was set before the transaction), so nothing had ever surfaced the failure. Only the new FAULTED lifecycle test revealed the empty EventLog. The fix was `session.merge()` — the same pattern already used in `CaseStatusChangedHandler`.

A bug invisible to 327 tests, found by the 328th.

## casehub-blackboard: brainstorm to 390 tests

I wanted both the planning layer and the Stage lifecycle layer designed together, because the planning strategy only makes sense once you understand what it's reasoning over. We went through a full brainstorm — Stage as a logical collection of Workers, `PlanItem<T>` as a generic hierarchy accommodating Stage, Worker, and SubCase, `CasePlanModel` separate from `CaseDefinition` so pure choreography stays the default when the module is absent.

Several design questions surfaced mid-session. The PENDING/RUNNING split: dropped — the async cycle transitions to RUNNING on creation, so PENDING has no observable window. The sealed interface: can't seal across Maven module boundaries, so `PlanElement` is a non-sealed marker in `api/`. The `@Alternative` CDI selection: needs explicit `quarkus.arc.selected-alternatives` in test `application.properties` alongside a `beans.xml`.

The implementation ran as 13 tasks via subagent-driven development. Each task: failing test, implementation, green, spec review, commit. 59 new tests in `casehub-blackboard`, 390 total across all three modules. The `PlanningStrategyLoopControl` integration tests — single stage activation, explicit exit condition, mixed choreography and orchestration — all pass.

PR #49 now carries 19 commits. Five PRs open in `casehubio/engine`, all waiting for co-owner review.
