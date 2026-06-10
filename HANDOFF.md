# Handoff — 2026-06-10

**Head commit (engine):** 5c3d38b4 — feat(panels): user-defined panels, listenPanel filtering, quality fixes — Closes #80, #81
**Head commit (workspace):** c80d925 — archive(issue-80-typed-casefile-panels): move plans to attic
**Both repos on:** main
**PR #467:** open — https://github.com/casehubio/engine/pull/467

## What Changed This Session

**#80 + #81 (CaseContext panels) — closed.** Full panel architecture delivered: `ReadablePanel`/`WritablePanel` interface hierarchy, `WritablePanelImpl` with `freeze()`, `CaseContextImpl` restructured to panel map. `asJsonNode()` now returns full panel document — all JQ expressions migrated to `.working.key` prefix (47 test files). Semantic panel (definition defaults + call-site augmentation), episodic panel intra-case (EventLog) + inter-case (`ReactiveCaseMemoryStore`), panel-aware recovery (`fromPanelDocument()`). User-defined panels + `listenPanel` binding subscription.

**Key quality fixes from review:** `deepCopyMap` shallow list bug (episodic workers list aliasing), `EpisodicPanelUpdater` R-M-W race → `engineUpdate()` atomic pattern, `CaseContextChangedEvent` `contextSnapshot` changed to `CaseContext` (eliminates milestones/goals asymmetry).

**Squash:** 30 → 18 commits. Fork push done. PR open to upstream.

## Immediate Next Step

Merge PR #467 or wait for review. While waiting: start `issue-446-working-memory-bridge` for the Drools `WorkingMemoryBridge` that depends on panels.

## What's Left

- PR #467: awaiting review/merge
- ledger#134: pre-existing `LedgerEntry.tenancyId` field shadowing blocks `@QuarkusTest` suites — fix in ledger repo before next engine `@QuarkusTest` work
- engine#466: review `casehub-platform` compile scope in `runtime/pom.xml`

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#446 | WorkingMemoryBridge — typed Drools facts from named panels | M | Med | Depends on #80/#81 (now done) |
| engine#5 | DroolsExpressionEvaluator | M | Med | Depends on #289 (done) |
| engine#207 | RulesRouter + RULES_DECISION lineage | L | Med | Final Drools piece — depends on #446 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#448 | Worker(Plan.of(...)) function type | M | Med | Plan-based execution Phase 1 |
| engine#465 | Validate panel event model serves Drools re-fire triggers | XS | Low | Before starting #446 |

## Key References

- PR #467: https://github.com/casehubio/engine/pull/467
- Spec: `proj/docs/specs/2026-06-09-casefile-panels-design.md` (rev 6)
- Blog: `wksp/blog/2026-06-10-mdp01-the-flat-map-that-grew-three-dimensions.md`
- Follow-up issues: engine#464 (panel naming), engine#465 (Drools events), engine#466 (pom scope), ledger#134 (field shadowing)
