# Handoff — 2026-05-29

**Head commit (engine):** b13f5da — docs(claude-md): add ProvisionResult SPI return type
**Head commit (workspace):** 4b704fa — docs: session handover 2026-05-29
**Branch:** `issue-392-sxs-batch` (engine) / `issue-392-sxs-batch` (workspace)

## What Changed This Session

PR#394 merged. Branch `issue-392-sxs-batch` started — 5 commits so far:
- #392 ✓ — disabled-path guard test for CaseLedger + WorkerDecision captures
- #389 ✓ — WorkerStarted CaseLifecycleEvent with causedByEntryId after provisioning
- #393 ✓ — await CaseCompleted CDI delivery in CaseStatusChangedHandler

Cross-module scan: all previous internal blockers resolved (#273, #377, platform#17 all closed).

## Immediate Next Step

Continue `issue-392-sxs-batch` — pick engine#395 first (Flyway path fix, XS·Low, unblocks AML).

## Cross-Module

**We're blocking:**
- `casehub-aml` — engine#395 (Flyway migrations at wrong classpath path) · XS · Low ← fix first, AML can't wire engine-ledger without it
- `casehub-aml` — engine#396 (CDI ambiguity when engine-ledger on classpath breaks AML tests) · S · Med

**Blocked by:** none — all cross-module deps resolved (engine#273, engine#377, platform#17 all closed).

## What's Left — S/XS batch (issue-392-sxs-batch)

In progress (this branch):
- engine#395 — fix: Flyway migrations path → `db/engine-ledger/migration/` · XS · Low [BLOCKS AML]
- engine#399 — bug: callerRef passed as assigneeIdOverride in handleTemplateMode · XS · Low
- engine#397 — fix: await fireAsync in 5 remaining lifecycle event handlers · S · Low
- engine#396 — bug: CDI ambiguity when adding engine-ledger to AML tests · S · Med [BLOCKS AML]
- engine#400 — feat: route WorkItem escalation events as case context signals · S · Med

## What's Left — larger work

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med ← NOW UNBLOCKED (#273 closed)
- engine#398 — bug: HumanTask completion silently dropped after JVM restart · M · Med
- engine#383 — oversight response loop: COMMAND re-triggers agent routing · M · Med ← NOW UNBLOCKED (#377 closed)
- engine#384 — PlanItem state during escalation (ESCALATING?) · M · Med ← NOW UNBLOCKED (#377 closed)
- engine#387 — humanTask: dynamic candidateGroups from case context · M · Med
- engine#299 — multi-tenancy foundation: tenancyId enforcement · L · High ← NOW UNBLOCKED (platform#17 closed)
- engine#389 — fire WorkerStarted after provisioning ✓ done this session
- ledger#100 — sequence race under READ COMMITTED (pre-existing) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#395 | Flyway path scoping fix | XS | Low | Blocks AML — do first |
| engine#399 | callerRef as assigneeIdOverride bug | XS | Low | — |
| engine#397 | await fireAsync in 5 remaining handlers | S | Low | Copy pattern from #393 |
| engine#396 | CDI ambiguity: engine-ledger + AML tests | S | Med | Blocks AML |
| engine#400 | WorkItem escalation as case context signal | S | Med | — |
| engine#274 | BlackboardRegistry hydration on restart | M | Med | Unblocked |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| parent#87 | PLATFORM.md capability table stale | S | Low | — |
| parent#88 | PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider | S | Low | — |

## Key References

- Blog: `blog/2026-05-29-mdp02-unblocking-aml.md`
- Protocol: PP-20260529-4783b2 — garden casehub/ledger-sequence-cross-subtype-query.md
- AML tracker: casehubio/aml#14, casehubio/aml#9 (Layer 6 now unblocked by #382+#390 merge)
