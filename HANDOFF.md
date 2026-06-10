# Handoff — 2026-06-10

**Head commit (engine):** a1ae3523 — fix(runtime): exclude CaseLedgerEventCapture from runtime test CDI
**Head commit (workspace):** 5baea5a — docs: add diary entry 2026-06-10-mdp02
**Both repos on:** main
**PRs #462 and #467:** MERGED ✓

## What Changed This Session

**#289 (expression evaluator) and #80/#81 (panels) — both merged.** Session started from two open PRs with conflicts. Resolved conflicts, fixed CI failures, squashed both branches, merged in order (#462 first, then #467).

**CI failures uncovered along the way:**
- `casehub-ledger` SNAPSHOT post-20260529 added `LedgerSequenceAllocator` requiring `ledger_subject_sequence` table. Engine runtime tests don't run ledger Flyway migrations → CDI observer fails silently → cascading test timeouts. Fix: exclude `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` from CDI in `runtime/src/test/resources/application.properties`.
- `LedgerEntry.tenancyId` field shadowing: both `CaseLedgerEntry` and `WorkerDecisionEntry` had duplicate fields rejected by the new `LedgerProcessor` validator. Removed from subclasses and V2000/V2001 SQL.
- `DefaultOutcomeRecorder`/`BlockingToReactiveOutcomeRecorder` needed CDI exclusion in ledger test `application.properties` — inject `CurrentPrincipal` not on test classpath.
- `applyTopLevelChanges` recovery bug: after panels, diff keys are panel names. `CaseContext.set("working", Map)` went through flat API and stored "working" as a nested key instead of replacing the panel. Fix: `ctxImpl.writablePanel(key).clear().setAll(afterMap)`.
- Several JQ expressions missed in the panels migration: `candidateGroupsExpression`, `Milestone.entryCriteria`, `WorkerScheduleDedupTest.inputSchema`.

## Immediate Next Step

Panels and expression evaluator are on main. Start `engine#465` (validate panel event model serves Drools re-fire triggers) then `engine#446` (WorkingMemoryBridge). Run `/work` to begin.

## What's Left

- ledger#134: pre-existing `LedgerEntry.tenancyId` field shadowing — now FIXED in engine. No further engine action needed; ledger repo may need its own cleanup.
- engine#466: review `casehub-platform` compile scope in `runtime/pom.xml` · XS · Low — still open, not urgent

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#465 | Validate panel event model serves Drools re-fire triggers | XS | Low | Do before #446 |
| engine#446 | WorkingMemoryBridge — typed Drools facts from named panels | M | Med | Unblocked |
| engine#5 | DroolsExpressionEvaluator | M | Med | Depends on #289 (done) |
| engine#207 | RulesRouter + RULES_DECISION lineage | L | Med | Final Drools piece — depends on #446 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#448 | Worker(Plan.of(...)) function type | M | Med | Plan-based execution Phase 1 |

## Key References

- Blog: `wksp/blog/2026-06-10-mdp02-the-database-that-wasnt-there-yesterday.md`
- Protocol: PP-20260610-18a084 (runtime test CDI exclusion) · PP-20260610-ecc2b2 (CASE_STARTED panel format)
- Garden entries: GE-20260610-1c73c1 (SNAPSHOT cascade), GE-20260610-c003ba (Surefire ClassSelector null)
