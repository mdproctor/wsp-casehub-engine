# Handoff — 2026-05-29

**Head commit (engine):** e12491b — feat: LangChain4jAgentEmbeddingProvider
**Head commit (workspace):** f6d0d15 — docs: session handover 2026-05-29 — engine#379,380,388,382,390 complete on sxs-batch branch
**Branch:** issue-382-sxs-batch (both repos, pushed)

## What Changed This Session

**PR#394 open on casehubio/engine — all S/XS issues complete.**

Branch `issue-382-sxs-batch` — 8 commits, squashed and pushed:

- **#379, #380** closed — already fixed in source (stale SNAPSHOT jars)
- **#388** — `TrustCandidateClassifier.decide()` → instance method; NaN sentinel removed; eidos-api dep comment; cosineSimilarity Javadoc
- **#382** — `TrustRoutingPolicy` + `TrustRoutingPolicyProvider` moved to `casehub-engine-api` (AML unblocked)
- **#390** — `WorkerDecisionEntry` JOINED ledger subclass; `WorkerDecisionEvent` CDI event; V2001 migration; actorId fix on CaseLifecycleEvent for worker completion (AML unblocked)
- **#281, #298** closed — already implemented/fixed
- **#381** — `CaseMemoryObserver` auto-captures terminal case events into `CaseMemoryStore`
- **#385** — `EmbeddingCache` LRU wired into `SemanticAgentRoutingStrategy`
- **#386** — `LangChain4jAgentEmbeddingProvider` via Quarkus `EmbeddingModel`

Filed: engine#392 (disabled-path test), ledger#100 (sequence race).

Infra note: Podman was at 2GB RAM — increased to 4GB. Socket path in `~/.testcontainers.properties` may need updating after future Podman restarts. Permanent fix: `sudo /opt/homebrew/Cellar/podman/5.8.2/bin/podman-mac-helper install`.

## Immediate Next Step

Wait for PR#394 review and merge. Next work item: **engine#274** — BlackboardRegistry hydration from PlanItemStore on restart (M·Med).

## What's Left

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med
- engine#383 — Oversight response loop: COMMAND → re-trigger routing · M · Med
- engine#384 — PlanItem state during escalation (ESCALATING state?) · M · Med
- engine#392 — test: no disabled-path test for WorkerDecisionEventCapture · XS · Low
- parent#87 — PLATFORM.md capability table stale · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | After PR#394 merge |
| engine#383 | Oversight response loop: COMMAND re-triggers routing | M | Med | After PR#394 merge |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |
| parent#88 | PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider | S | Low | — |

## Key References

- PR: casehubio/engine#394
- Branch: `issue-382-sxs-batch`
- Filed: engine#392, ledger#100
