# Handoff — 2026-06-09

**Head commit (engine):** f1dd5edd — fix: update stubs for casehub-ledger/qhorus tenancyId API
**Head commit (workspace):** d61c5e1 — feat: promote blog from issue-289-expression-evaluator-factory
**Both repos on:** main
**PR #462:** open — https://github.com/casehubio/engine/pull/462

## What Changed This Session

**#289 (ExpressionEvaluatorFactory SPI) — closed.** Three spec critique rounds before implementation. Key design: extended `ExpressionEngine` with `create(String)` + `supportsStringCreation()` rather than a new factory interface. `ExpressionEngineRegistry` moved `common/spi/` → `api/engine/`; `@YamlMapper` moved to `api/marshaller/`. `CaseDefinitionYamlMapper` static workaround deleted; `YamlCaseHub` now injects directly. `ObjectMapperInjector` deleted (was firing at CDI priority 2500, after definitions loaded). `expressionLang` YAML field added.

**#434 (classifier-throws fail-safe integration test) — closed.** `throwOnClassify` flag added to `CapturingClassifier`; one new test in `ActionGateIntegrationTest`.

**Pre-existing API fix-ups:** casehub-ledger and casehub-qhorus 0.2-SNAPSHOT added tenancyId param to all repo/event methods. Fixed stubs and callers across runtime, blackboard, work-adapter, resilience, actor-state, ledger modules.

**ADR-0009:** expressionLang granularity — per-definition vs per-expression. CNCF SW 1.0 alignment.

## Immediate Next Step

Start `issue-80-typed-casefile-panels` for #80 + #81. **Brainstorm-first, no code.** M/High — these need platform coherence review before implementation.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- engine#433: persist `pendingActionGate` in `CaseInstanceEntity` (restart resilience) · M · Med
- PR #462: awaiting review/merge
- Ledger QuarkusTest CDI startup issue (CurrentPrincipal unsatisfied) — pre-existing, tracked separately

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#80 + #81 | Typed CaseContext panels | M | High | Brainstorm-first; prerequisite for WorkingMemoryBridge |
| engine#446 | WorkingMemoryBridge | M | Med | Depends on #80/#81 |
| engine#5 | DroolsExpressionEvaluator | M | Med | Depends on #289 (now done) — but Drools integration design unclear, file GH issue before #5 |
| engine#207 | RulesRouter + RULES_DECISION lineage | L | Med | Final Drools piece — depends on #446 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#448 | Worker(Plan.of(...)) function type | M | Med | Plan-based execution Phase 1 |

## Key References

- PR #462: https://github.com/casehubio/engine/pull/462
- Design spec: `wksp/specs/2026-06-09-expression-evaluator-factory-design.md`
- ADR-0009: `docs/adr/0009-expression-lang-granularity.md`
- Diary: `wksp/blog/2026-06-09-mdp01-the-factory-that-wasnt.md`
