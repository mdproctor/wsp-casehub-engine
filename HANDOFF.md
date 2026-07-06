# Handoff — 2026-07-06

## What's Done

**#478 implementation complete** on branch `issue-478-case-retriever-routing-bridge` (20 commits). CBR Retrieve→Reuse bridge: sealed `FeatureExtractor` (JQ/lambda), `CbrConfig` on `CaseDefinition`, `RetrievedExperience`/`ExperiencePlanStep` engine-owned types, routing context enrichment (`tenancyId` gap fix + `experiences` field on all 3 contexts), `CbrRetrievalService` with failure recovery, YAML `cbr:` schema + mapper, wired into `CaseContextChangedEventHandler` + `DefaultWorkOrchestrator`. Design review (4 rounds, 16 issues, all resolved). 47+ tests pass. Full suite green (1 pre-existing flaky in work-adapter — set ordering, unrelated).

**Blocker:** 2 integration tests `@Disabled` pending #675 — `BlockingToReactiveCbrBridge` (`@DefaultBean`) resolves delegate to `NoOpCbrCaseMemoryStore` instead of `InMemoryCbrCaseMemoryStore`. Root cause: Quarkus ARC resolves `@DefaultBean` injection points at build time before `@Alternative` activation. Fix designed: switch `CbrRetrievalService` to blocking `CbrCaseMemoryStore` + `runSubscriptionOn()`, use `getEnabledAlternatives()` with full alternative re-declaration. Garden entry: GE-20260706-abaddc.

## Cross-Module

**devtown needs follow-up** — #667: two devtown classes extend renamed engine implementations.

## What's Left

- #675 — fix CBR integration tests (blocking store + getEnabledAlternatives) · S · Med — **gates merge of #478**
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
