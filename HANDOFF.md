# Handoff — 2026-07-07

## What's Done

**#478, #675, #663, #668 all landed on main** via branch `issue-478-case-retriever-routing-bridge` (16 squashed commits). CBR Retrieve→Reuse bridge complete: `CbrRetrievalService`, sealed `FeatureExtractor`, `CbrConfig` on `CaseDefinition`, routing context enrichment (`experiences` field on all 3 contexts), YAML `cbr:` schema, 15 tests. #675 fixed (blocking store bypasses `@DefaultBean` bridge). #663 fixed (interface-based delegation for all 5 reactive in-memory repos). #668 `TrustGatedAttestationPolicy` in engine-ledger. Garden entry GE-20260707-f3bece captures the delegate injection gotcha.

## Cross-Module

**devtown needs follow-up** — #667: two devtown classes extend renamed engine implementations.

## What's Left

- #654 — populate CaseMetaModel definition column (paused on stack) · S · Low
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #666 — consolidate WorkerRetryExhaustionHandler + PlanItemFaultHandler · S · Med
- #669 — SubCaseMofNOutputMappingTest suite interaction · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #671 | Case-lifetime CBR caching for high-frequency ticks | S | Med | QuarkMind blocked until this ships |
| #672 | Feature-level similarity breakdown in RetrievedExperience | S | Med | |
| #673 | CbrConfig validation at registration time | XS | Low | |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
| #655 | Vocabulary validation for types/labels | M | Med | |
| #667 | Devtown cross-repo rename propagation | S | Low | |
