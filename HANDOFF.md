# Handover — issue-995-unified-judgment-target

**Branch:** `issue-995-unified-judgment-target`
**Covers:** #995, #999, #1000, #957
**Date:** 2026-08-29

## What happened

This session delivered three governed yield issues and started a fourth:

1. **#996 (JudgmentScheduler SPI) — CLOSED.** JudgmentTarget sealed permit, JudgmentScheduler in common/spi/, NoOp default, YAML parsing, dispatch handler, completion/expiry handlers. Landed as `9d5b294b`.

2. **#997 (JudgmentVerifier SPI) — CLOSED.** JudgmentVerifier extends NamedStrategy, VerificationResult sealed type, AcceptAllVerifier (@DefaultBean), EvidencePresenceVerifier, verification gate in JudgmentCompletedHandler, escalation path. Landed as `32876425`.

3. **#998 (judgment ledger events) — CLOSED.** Four CaseHubEventType constants (YIELDED, RESPONDED, VERIFIED, ESCALATED) with metadata schemas. Landed with #996.

4. **#995 (unified JudgmentTarget) — IN PROGRESS.** Design complete, production code complete. HumanTaskTarget stripped from BindingTarget sealed permits, retained as scheduler-layer data carrier. RoutingConfig/HumanRoutingConfig sealed types added. JudgmentTarget expanded with title, outcomes, scope, priority, trustThreshold, escalatorStrategy, routingConfig. All 9 production switch sites updated. YAML parsing handles both `judgment:` with `human:` sub-block and legacy `humanTask:` block.

## Key decision

**D1: Unified JudgmentTarget with RoutingConfig** — first-principles analysis. Every yield separates WHAT (prompt/evidence on target), WHO (routing hints on RoutingConfig), HOW VERIFIED (verifier/escalator on target). HumanTaskTarget mixed all three. Sealed RoutingConfig extensible for future caller types. DelegatingJudgmentScheduler bridges to existing HumanTaskScheduler.

## Resume from

**Remaining on this branch (Batch 1 test migration + Batches 2-5):**
- Test migration: ~8 test files still reference `HumanTaskTarget` or `.humanTask()` builder — `HumanTaskTargetDispatchTest`, `HumanTaskTypedContextTest`, `CbrCaseRetainObserverTest`, remaining `CaseDefinitionYamlMapperTest` field access patterns
- Task 2: `DelegatingJudgmentScheduler` + unified `publishJudgmentSchedule()`
- Task 3: `JudgmentEscalator` SPI (#999)
- Task 4: `JudgmentNodeExecutor` + SWF `casehub:judgment` (#1000)
- Task 5: React module integration tests (#957)

**Cross-repo work slot** needed after engine changes land — 8 repos (engine, work, clinical, devtown, life, soc, examples, fsitrading) in one IntelliJ workspace for semantic refactoring. Never find-and-replace.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-995-unified-judgment-target/2026-08-29-unified-judgment-target-design.md` |
| Decisions | `specs/issue-995-unified-judgment-target/decisions.md` |
| Plan | `plans/2026-08-29-unified-judgment-target.md` |
| .plan | `.plan` (queue position 0/1, active #995) |
