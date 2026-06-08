# BlackboardRegistry Lifecycle Design

**Issue:** casehubio/engine#404  
**Date:** 2026-06-08  
**Status:** Design — no implementation

---

## Problem

`BlackboardRegistry` currently evicts only at terminal state (COMPLETED/FAULTED/CANCELLED).
A case waiting six months for a human approval holds its `CasePlanModel` in the registry for
that entire duration. For deployments with thousands of concurrent long-running cases this is
a real memory problem. Additionally, restart recovery is incomplete: DELEGATED PlanItems
hydrate on demand, but RUNNING Quartz workers have no persistent record and are invisible
after restart.

---

## Case State Map

```
RUNNING ──► WAITING ──► RUNNING   (external trigger re-enters)
   │            │
   │            └──► SUSPENDED ──► RUNNING
   │
   ├──► COMPLETED  (terminal)
   ├──► FAULTED    (terminal)
   └──► CANCELLED  (terminal)
```

**RUNNING:** At least one Quartz job is active OR the case is being re-evaluated after a
context change. PlanItems in RUNNING or PENDING state.

**WAITING:** All active workers are DELEGATED (HumanTask) or the case is blocked on an
external event (timer, Qhorus message, signal). This is the idle state — no Quartz threads
are executing against this case.

**SUSPENDED:** Admin pause. Semantically equivalent to WAITING for registry purposes — no
active workers.

**Terminal (COMPLETED/FAULTED/CANCELLED):** No further execution. Registry entry can be
released.

---

## Touch Point Map

Every trigger that causes a handler to look up the registry:

| Trigger | Handler | Registry call | At state |
|---------|---------|---------------|----------|
| `WORKER_EXECUTION_FINISHED` | `PlanItemCompletionHandler` | `getPlanItemId(caseId, workerName)` | RUNNING → transition |
| `WORKER_RETRIES_EXHAUSTED` (Quartz) | `WorkerRetryExhaustionHandler` / `PlanItemFaultHandler` | `getPlanItemId(caseId, workerId)` | RUNNING → FAULTED |
| WorkItem COMPLETED/REJECTED/EXPIRED/CANCELLED | `WorkItemLifecycleAdapter` | `registry.get(caseId)` | WAITING → RUNNING |
| WorkItem group lifecycle | `WorkItemLifecycleAdapter` | `registry.get(caseId)` | WAITING → RUNNING |
| `ACTION_GATE_APPROVED` | `ActionGateApprovedHandler` | (indirect via CasePlanModel) | RUNNING |
| `ACTION_GATE_REJECTED` | `ActionGateRejectedPlanItemHandler` | `getPlanItemId(caseId, workerId)` | RUNNING |
| `ACTION_GATE_EXPIRED` | `ActionGateExpiredPlanItemHandler` | `getPlanItemId(caseId, workerId)` | RUNNING |
| `CONTEXT_CHANGED` | `CaseContextChangedEventHandler` | (policy loop, via `getOrCreate`) | RUNNING |
| Qhorus SIGNAL_RECEIVED | `CaseContextChangedEventHandler` | (policy loop) | WAITING → RUNNING |
| Sub-case completion | `SubCaseCompletionService` / `PlanItemCompletionHandler` | `getPlanItemId(caseId, childCaseId)` | WAITING → RUNNING |
| Admin resume (SUSPENDED → RUNNING) | `CaseStatusChangedHandler` publishes CONTEXT_CHANGED | (policy loop) | SUSPENDED → RUNNING |

**Pattern:** Every re-entry touch point calls either `getPlanItemId` (needs completionIndex) or
`registry.get(caseId)` (needs CasePlanModel with DELEGATED PlanItems restored).

---

## What must be in the registry for each touch point

| Touch point | CasePlanModel | completionIndex |
|-------------|---------------|-----------------|
| WorkItem lifecycle | ✅ DELEGATED PlanItems restored by lazy hydration | ❌ not used — routes via `callerRef` (planItemId embedded) |
| WorkItem group lifecycle | ✅ model present | ❌ not used — routes via `callerRef` |
| Quartz WORKER_EXECUTION_FINISHED | ✅ RUNNING PlanItems in model | ✅ workerName→planItemId mapping |
| ACTION_GATE_REJECTED/EXPIRED | ✅ model present | ✅ workerId→planItemId mapping |
| Sub-case completion | ✅ model present | ✅ childCaseId→planItemId mapping |
| CONTEXT_CHANGED (re-entry) | ✅ model present (configurer re-applies) | rebuilt at next schedule |

---

## completionIndex — reconstructibility

`completionIndex` maps `workerName → planItemId`. It is populated at **schedule time** by
`PlanningStrategyLoopControl.indexSelectedForCompletion(caseId, pi.getWorkerName(), pi.getPlanItemId())`.

`workerName == bindingName` (confirmed: `pi.getWorkerName()` is the binding name from
`CaseDefinition`). `PlanItemRecord` carries `bindingName`. Therefore:

- **DELEGATED PlanItems:** completionIndex is fully reconstructible from
  `PlanItemStore.findDelegated(caseId)` using `r.bindingName()` as the key.
- **RUNNING Quartz PlanItems:** NOT persisted today. Cannot be rebuilt after eviction.
  Requires `workerName` in `PlanItemRecord` AND a save call at schedule time.

This is the core constraint for strategy selection.

---

## Strategy Analysis

### Strategy A — Evict at terminal only (current)

Eviction trigger: COMPLETED, FAULTED, CANCELLED.

**What's needed:** nothing beyond current implementation.

**Memory profile:** unbounded for long-running cases. A case waiting 6 months for human
approval holds its CasePlanModel for 6 months. Scales linearly with concurrent case count
and case duration. Unacceptable for deployments with high case volume and HumanTask-heavy
workflows.

**Restart recovery:** incomplete. DELEGATED PlanItems hydrate on demand. RUNNING Quartz
workers are invisible after restart (no persisted record).

**Verdict:** Correct for low-volume, short-duration deployments. Untenable for the target
workload (long-running compliance cases, thousands concurrent).

---

### Strategy B — Stateless-on-rest (recommended)

Eviction trigger: case enters WAITING or SUSPENDED — the idle state where no Quartz threads
are active.

At WAITING, all active PlanItems are DELEGATED. The CasePlanModel can be evicted. When the
next trigger arrives (WorkItem lifecycle, Qhorus signal, timer), re-hydrate from
`PlanItemStore`.

**Phase 1 — Evict at WAITING (implementable now, zero persistence cost):**

- `CaseEvictionHandler` adds WAITING and SUSPENDED to the eviction trigger set.
- `BlackboardRegistry.get(UUID)` lazy hydration already loads DELEGATED PlanItems.
- `completionIndex` does NOT need rebuilding for WAITING re-entry. `WorkItemLifecycleAdapter`
  routes via `CallerRef.parse(workItem.callerRef)` which embeds `planItemId` directly — it
  calls `registry.get(caseId)` to get the CasePlanModel and then `plan.getPlanItem(planItemId)`
  by ID. The completionIndex is only needed for Quartz RUNNING workers, which are absent
  at WAITING.
- No new persistence fields needed.

Constraint: safe ONLY because at WAITING, all active workers are DELEGATED and the
completionIndex is not used for their re-entry path. This invariant must be maintained — it
is currently guaranteed by the state machine (a case enters WAITING when there are no
scheduled Quartz jobs and all triggered PlanItems are DELEGATED).

**Phase 2 — Full coverage: persist RUNNING PlanItems (future issue):**

Currently RUNNING Quartz PlanItems are not persisted. After a JVM restart, completion events
for in-flight Quartz workers cannot be routed to their PlanItem.

Required changes:
- Add `workerName` to `PlanItemRecord`.
- Save a PlanItem record with `status=RUNNING` at schedule time
  (`PlanningStrategyLoopControl.indexSelectedForCompletion`).
- Extend `BlackboardRegistry` hydration to load ALL non-terminal PlanItems
  (`PlanItemStore.findByCaseId`) instead of `findDelegated` only.
- Rebuild `completionIndex` from `workerName` field on hydration.

With Phase 2, RUNNING eviction becomes safe: evict on any trigger (e.g., LRU), re-hydrate
from full `findByCaseId` on next access.

**Memory profile:** At WAITING, the registry entry is released. Re-entry cost: one blocking
`PlanItemStore.findDelegated` call (already exists). For cases that cycle RUNNING→WAITING
frequently, each WAITING entry costs ~one DB query at re-entry. This is negligible compared
to 6 months of in-memory state.

**Correctness constraint:** The eviction point must be after all RUNNING → DELEGATED
transitions commit. `CaseEvictionHandler` already listens on `CASE_STATUS_CHANGED`, which
fires after the status write commits. This is safe.

**Restart completeness (Phase 1):** After restart, a case in WAITING state re-hydrates
correctly on next WorkItem event. RUNNING Quartz workers from before restart remain
unrecoverable (unchanged from today — Phase 2 fixes this).

---

### Strategy C — LRU soft eviction

Evict least-recently-used CasePlanModels above a configurable threshold (e.g.,
`casehub.engine.registry.max-entries=1000`). Re-hydrate on access miss.

**What's needed:** `BlackboardRegistry` wraps `ConcurrentHashMap` with an LRU eviction
policy (e.g., Caffeine). Phase 2 persistence prerequisites for safe RUNNING eviction.

**Memory profile:** bounded at `max-entries`. Predictable heap footprint regardless of case
duration.

**Correctness:** safe only after Phase 2 (RUNNING PlanItems persisted). Without Phase 2,
evicting a case with an active Quartz job means the completion event's `completionIndex`
lookup returns empty and the PlanItem is never marked COMPLETED.

**Operational model:** requires tuning `max-entries` per deployment. Eviction is
non-deterministic from the case's perspective — harder to reason about than state-based
eviction.

**Verdict:** A good backstop for unbounded growth, but secondary to state-based eviction.
Best used as a safety net on top of Strategy B, not instead of it.

---

## Recommendation

**Adopt Strategy B in two phases.**

Phase 1 is implementable today: add WAITING and SUSPENDED to the eviction trigger set in
`CaseEvictionHandler`. Zero persistence schema changes. Zero new SPI methods. The invariant
"at WAITING, all active workers are DELEGATED" is structurally enforced by the state machine.

Phase 2 (tracked separately) persists RUNNING PlanItems at schedule time, enabling full
restart recovery and opening the door to LRU as a configurable safety cap.

**Do not implement LRU without Phase 2.** Evicting a RUNNING case without persisted
completionIndex corrupts completion routing. LRU without persistence is silent data loss.

---

## Configuration (Phase 2+)

```properties
# eviction strategy: waiting (default), lru, terminal-only
casehub.engine.registry.eviction-strategy=waiting

# LRU cap (only meaningful when strategy=lru)
casehub.engine.registry.max-entries=10000
```

Strategy `waiting` is the default. `terminal-only` preserves current behavior for
deployments that accept unbounded memory in exchange for zero hydration cost.

---

## Persistence Prerequisites Summary

| Requirement | Needed for | Status |
|-------------|-----------|--------|
| `PlanItemStore.findDelegated(UUID)` | Phase 1 hydration | ✅ exists |
| `PlanItemRecord.bindingName` | completionIndex rebuild from DELEGATED | ✅ exists |
| `PlanItemRecord.workerName` | completionIndex rebuild from RUNNING | ❌ not yet |
| `PlanItemStore.findByCaseId(UUID, tenancyId)` | Phase 2 full hydration | ✅ exists |
| Save RUNNING PlanItem at schedule time | Phase 2 | ❌ not yet |
| Stage state persistence | Phase 2 (stages rebuilt by configurer on re-entry) | ❌ not yet |

**Stage state note:** stages are not persisted today — they are rebuilt by configurers when
a case re-enters. This is consistent with stateless-on-rest: the configurer re-applies the
CaseDefinition's stage structure on re-entry. For correctness, the configurer must be
idempotent with respect to PlanItem state. Verify this invariant before enabling Phase 1 on
stage-heavy case definitions.

---

## Follow-on Issues

- Phase 2 implementation: persist RUNNING PlanItems, add `workerName` to `PlanItemRecord`
- LRU safety cap: add Caffeine to `BlackboardRegistry` (gated on Phase 2)
- Stage re-entry idempotency: audit configurers for non-idempotent stage setup
