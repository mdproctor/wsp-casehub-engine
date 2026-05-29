# Handoff — 2026-05-29

**Head commit (engine):** e12491b — feat: LangChain4jAgentEmbeddingProvider
**Head commit (workspace):** 9779a39 — feat: promote blog from issue-382-sxs-batch
**Both repos on:** main

## What Changed This Session

**`issue-382-sxs-batch` closed — PR#394 open on casehubio/engine.**

All S/XS issues resolved (8 commits):
- #388 (XS) — TrustCandidateClassifier quality fixes
- #382 (S) — TrustRoutingPolicy + TrustRoutingPolicyProvider moved to casehub-engine-api (**AML unblocked**)
- #390 (S) — WorkerDecisionEntry JOINED ledger subclass + WorkerDecisionEvent CDI event + V2001 migration (**AML unblocked**)
- #381 (S) — CaseMemoryObserver auto-captures terminal case events
- #385 (S) — EmbeddingCache LRU wired into SemanticAgentRoutingStrategy
- #386 (S) — LangChain4jAgentEmbeddingProvider via Quarkus EmbeddingModel
- #379, #380, #281, #298 — closed as already fixed/implemented

Garden: 3 entries (podman restart socket variant, @Transactional pool exhaustion, LinkedHashMap LRU).
Protocol: PP-20260529-4783b2 — ledger-sequence-cross-subtype-query.
Blog: `blog/2026-05-29-mdp02-unblocking-aml.md` published.

Infra: Podman bumped to 4GB RAM. `~/.testcontainers.properties` may need updating after future restarts. Permanent fix: `! sudo /opt/homebrew/Cellar/podman/5.8.2/bin/podman-mac-helper install`.

## Immediate Next Step

Wait for PR#394 review/merge. Next work: **engine#274** — BlackboardRegistry hydration from PlanItemStore on restart (M·Med).

## What's Left

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med
- engine#383 — Oversight response loop: COMMAND → re-trigger routing · M · Med
- engine#384 — PlanItem state during escalation (ESCALATING state?) · M · Med
- engine#392 — test: no disabled-path test for WorkerDecisionEventCapture · XS · Low
- parent#87 — PLATFORM.md capability table stale · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider · S · Low
- ledger#100 — sequence race under READ COMMITTED (pre-existing, tracked in casehub-ledger) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | After PR#394 merge |
| engine#383 | Oversight response loop: COMMAND re-triggers routing | M | Med | After PR#394 merge |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |
| parent#88 | PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider | S | Low | — |

## Key References

- PR: casehubio/engine#394
- Blog: `blog/2026-05-29-mdp02-unblocking-aml.md`
- Protocol: PP-20260529-4783b2 — garden casehub/ledger-sequence-cross-subtype-query.md
- Filed: engine#392, ledger#100
