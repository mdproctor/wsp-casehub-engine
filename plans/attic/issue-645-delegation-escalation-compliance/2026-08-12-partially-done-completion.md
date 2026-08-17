# Partially-Done Issue Completion Plan

Generated: 2026-08-12
Source: issue audit of 131 open engine issues — 37 closed (22 done + 15 obsolete/effectively-done), 14 partially-done items remain.

## Overview

After auditing all 131 open engine issues against the codebase, 14 issues
remain in a partially-done state. This plan organises them into three tiers
by effort and actionability.

---

## Tier 1 — Quick Wins (S effort, closable with focused sessions)

### #656 Instance-level types and labels on CaseInstance
- **Done:** `Set<String> labels` exists on CaseInstance (added for queue module #730)
- **Remaining:** Add `Set<String> types` field (same pattern as labels)
- **Files:** `CaseInstance`, `CaseInstanceEntity`, YAML mapper
- **Effort:** S — mechanical, same pattern as labels

### #670 Epic: Eidos behavioral contracts and match-quality adoption
- **Done:** 3/4 children closed (#638, #647, #632). #639 now closed (obsolete).
- **Remaining:** Only #645 (observe delegation and escalation compliance) — add `ComplianceDimension.DELEGATION` and `ComplianceDimension.ESCALATION` to `BehavioralComplianceRecorder`
- **Effort:** S — extend existing recorder

### #833 Epic: ACL engine integration
- **Done:** 4/5 tasks complete (identity propagation, REST enforcement, worker rights, tenant isolation)
- **Remaining:** Only #855 (listCases/listAll leak metadata when ACL active) — add ACL filtering to list endpoints in CaseService
- **Effort:** S — add filter logic to existing methods

### #212 Multi-instance WorkItem spawning (engine scope)
- **Done:** ActionGate M-of-N quorum (#810) uses `WorkItemCreator.createMultiInstance()`
- **Remaining:** General-purpose lineage threading, context distribution, completion-to-worker integration
- **Note:** Work-side infrastructure exists; engine integration is the gap
- **Effort:** M — but engine-specific scope is S (lineage threading)

---

## Tier 2 — Medium Effort (M, each needs a focused session)

### #22 SLA/SLO Tracking and Deadline Monitoring
- **Done:** Milestone-level SLA (`MilestoneSLATimeoutJob`, `MilestoneSLAViolatedEvent`)
- **Remaining:** Case-level SLA (#510 — timer-triggered binding for overall case deadline), goal-level SLA, metrics/observability
- **Depends on:** #660 (timer sentry type) for proper implementation
- **Effort:** M — new trigger type + SLA infrastructure

### #696 Multi-level recovery protocol
- **Done:** Level 1 (retry: `QuartzRetryService`, `RetryPolicies`), Level 2 partial (`PlanAdaptationEvaluator` + `OnFailureTrigger`)
- **Remaining:** Error classification (transient vs reasoning vs fundamental), unified escalation connecting the three levels, side-effect classification for retry safety
- **Effort:** M — design-heavy, needs spec

### #697 Plan versioning — immutable plan snapshots
- **Done:** `ExecutionSnapshotStore` with `DagPlanSnapshot` and `DagResultSnapshot` capture
- **Remaining:** Version IDs on snapshots, version sequence tracking, delta/rationale comparison, rollback, compliance linkage
- **Effort:** M — versioning semantics on existing infrastructure

### #108 Long-Running Case Management
- **Done:** Persistence (JPA), recovery (`WorkerExecutionRecoveryService`), lifecycle states (WAITING/SUSPENDED)
- **Remaining:** `CaseScheduler` for cross-session resume, `SlaBreachAction` enum, stale/breached case queries, business-calendar scheduling
- **Effort:** M — scheduling and query API

### #656 → already listed in Tier 1

---

## Tier 3 — Tracker Epics (update scope, not implementation)

These are parent epics where the remaining children are independently tracked.
They need scope updates rather than implementation work.

### #77 Epic: Blackboard Architecture Evolution
- **Status:** 2/6 closed (#80, #81). #82 now closed (obsolete). 3 remain (#78, #79, #83) — all labelled `future`
- **Action:** Update epic description to reflect current state; remaining children are future research items

### #102 Epic: casehub Ecosystem Use Cases
- **Status:** Some children closed. Most remain as future use-case descriptions
- **Action:** No code work — close when children close

### #201 Epic: Adaptive execution architecture
- **Status:** #203 done, #206 done, #202/#208 now closed (obsolete). Open: #204, #205, #207, #209, #210, #211
- **Action:** Update epic description — several children are now independently scoped

### #595 Epic: Execution Capability Models
- **Status:** 7/11 children done (#596-#602). 4 remain (#603-#606)
- **Action:** Update epic description; remaining models are future/research

### #104 RAG pipelines with large artefact sharing
- **Status:** Generic infra done (DataChannel, Exchange, DataRef). RAG-specific DSL not built
- **Action:** Re-scope — the engine provides generic infrastructure; RAG-specific types may not belong here

### #107 Elastic research teams — multi-agent mesh
- **Status:** Engine-side parallel activation, DataChannel, Exchange done. Qhorus peer mesh is out of scope
- **Action:** Re-scope or close — remaining work is Qhorus-side

### #113 Regulatory decision automation
- **Status:** Audit infra strong (ledger, gates, quorum). Decision-rationale capture layer missing
- **Action:** Separate issue for `DecisionRationale` if pursued

### #114 ReAct cycles with full auditability
- **Status:** Building blocks exist. ReAct-specific orchestration layer not built
- **Action:** Evaluate whether blocks patterns cover this; if so, close

### #116 Compliance and audit workflows
- **Status:** Ledger foundation done. Compliance annotation/query layer missing
- **Action:** Separate issue for compliance query API if pursued

### #230 Normative layer audit
- **Status:** `MessageType` on channels done. Systematic vocabulary application not done
- **Action:** Future/research — update scope

---

## Recommended Session Order

1. **#656** (types field) — XS, quick win, unblocks queue/label features
2. **#833/#855** (list metadata leak) — S, ACL correctness fix
3. **#670/#645** (delegation compliance) — S, extends existing recorder
4. **#22/#510** (case-level SLA) — M, new capability, high value
5. **#696** (recovery protocol) — M, design-heavy, needs spec first
6. **#697** (plan versioning) — M, infrastructure improvement
7. Tracker epic updates — batch update descriptions in one session
