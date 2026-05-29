# Handoff — 2026-05-29

**Head commit (engine):** 4498185 — feat: WorkerDecisionEntry — tamper-evident ledger entry per worker execution
**Head commit (workspace):** ff29364 — docs: session handover 2026-05-29 — add engine#382 + #390 (AML blocked)
**Branch:** issue-382-sxs-batch (both repos)

## What Changed This Session

**S/XS batch in progress on branch `issue-382-sxs-batch`.**

Completed so far (5 of 9):
- #379, #380 — closed as already fixed in source (SNAPSHOT jar stale, current source correct)
- #388 — TrustCandidateClassifier.decide() → instance method; NaN sentinel removed from test helper; eidos-api dep comment corrected; cosineSimilarity Javadoc fixed
- #382 — TrustRoutingPolicy + TrustRoutingPolicyProvider moved to casehub-engine-api (AML unblocked); TrustRoutingPolicyTest moved to api/src/test; FQN cleanup in TrustWeightedAgentStrategy
- #390 — WorkerDecisionEntry JOINED ledger subclass + WorkerDecisionEvent + WorkerDecisionEventCapture; CaseLedgerEventCapture.findLatestByCaseId → findLatestBySubjectId (cross-subtype sequence fix); CaseLifecycleEvent actorId for worker completion changed to "system"; extractCapabilityTag + resolveConflictStrategy unified via findMatchingCapabilityBinding; V2001 migration; 3 new integration tests

Filed during review:
- engine#392 — test: no disabled-path test for WorkerDecisionEventCapture
- ledger#100 — fix: ledger_entry sequence race under READ COMMITTED isolation (pre-existing, tracked in casehub-ledger)

## Immediate Next Step

Continue on `issue-382-sxs-batch`: **engine#281** — add `JpaReactivePlanItemStoreContractTest` + abstract `ReactivePlanItemStoreContractTest` base in engine-common.

## What's Left (this branch)

- engine#281 — JpaReactivePlanItemStoreContractTest + abstract base · S · Low
- engine#298 — Fix HumanTaskScheduleHandlerTest isolation failures (7 pre-existing) · S · Low
- engine#381 — CDI observer: auto-capture memories from case lifecycle events into CaseMemoryStore · S · Low
- engine#385 — Embedding cache for SemanticAgentRoutingStrategy · S · Low
- engine#386 — LangChain4j AgentEmbeddingProvider implementation · S · Low

## What's Next (post-branch)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| engine#383 | Oversight response loop: COMMAND from human re-triggers routing | M | Med | — |
| engine#384 | PlanItem state during escalation (ESCALATING state?) | M | Med | — |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |
| parent#88 | PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider | S | Low | — |

## Key References

- Branch: `issue-382-sxs-batch` (engine + workspace)
- Specs: `proj/docs/specs/2026-05-29-trust-routing-policy-api-move-design.md`, `proj/docs/specs/2026-05-29-worker-decision-entry-design.md`
- Filed: engine#392, ledger#100
- Prior PR: casehubio/engine#391 (merged — AgentRoutingStrategy reactive SPI)
