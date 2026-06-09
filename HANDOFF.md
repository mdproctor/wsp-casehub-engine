# Handoff — 2026-06-09

**Head commit (engine):** 05f3fd56 — test(#434): integration test for classifier-throws fail-safe
**Head commit (workspace):** 7265837 — chore: update covers to 289,434
**Branch:** issue-289-expression-evaluator-factory (open — covers #289, #434; ready to close)
**PR #451:** open — actor-state tests + design specs (awaiting review/merge)

## What Changed This Session

**#289 (ExpressionEvaluatorFactory SPI) — closed.**
- `ExpressionEngine` extended: `create(String expression)` default + `supportsStringCreation()` default
- `JQExpressionEngine` overrides both: `create()` returns `new JQExpressionEvaluator(expression)`, `supportsStringCreation()` returns `true`
- `ExpressionEngineRegistry` (moved from `common/spi/` → `api/engine/`): `create(expr, lang)` with type-assertion invariant; `assertLanguageSupported(lang)` without side effects
- `DefaultExpressionEngineRegistry`: implements both, enforces `evaluator.type() == expressionLang`
- `@YamlMapper` qualifier moved `runtime/internal/marshaller/` → `api/marshaller/` (new package)
- `casehub-engine-api` added as explicit dep in `scheduler-quartz/pom.xml`
- Schema: `expressionLang` field added to `CaseDefinition.yaml` (default `"jq"`)
- `CaseDefinitionYamlMapper`: static `yamlMapper`/`setObjectMapper()` deleted; 3-arg `load(InputStream, ObjectMapper, ExpressionEngineRegistry)` added; all 5 `new JQExpressionEvaluator()` call sites delegate to registry; `JQ_ONLY` anonymous registry for non-CDI path
- `YamlCaseHub`: injects `ExpressionEngineRegistry` and `@YamlMapper ObjectMapper` directly; no static workaround
- `ObjectMapperInjector` deleted (TODO resolved)
- Pre-existing fixes: `NoOpLedgerEntryRepository` (new tenancyId-scoped API), `QhorusMessageSignalBridgeTest` (new tenancyId param)

**#434 (classifier-throws fail-safe integration test) — closed.**
- Added `throwOnClassify` flag to `CapturingClassifier`
- New test `classifierThrows_failSafeGateRequired_caseRemainsRunning` in `ActionGateIntegrationTest`
- Confirmed `.onFailure().recoverWithItem()` path in `handleWithPlannedAction` fires correctly

**Design spec:** `specs/2026-06-09-expression-evaluator-factory-design.md` (in workspace, not promoted yet)

## Immediate Next Step

Close branch `issue-289-expression-evaluator-factory` (covers 289, 434 — both done). Then start `issue-80-typed-casefile-panels` for #80 + #81.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- engine#433: persist `pendingActionGate` in `CaseInstanceEntity` (restart resilience) · M · Med
- PR #451: awaiting review/merge

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#80 + #81 | Typed CaseContext panels | M | High | Brainstorm-first; prerequisite for WorkingMemoryBridge |
| engine#446 | WorkingMemoryBridge | M | Med | Depends on #80/#81 |
| engine#5 | DroolsExpressionEvaluator | M | Med | Depends on #289 (now done) |
| engine#207 | RulesRouter + RULES_DECISION lineage | L | Med | Final Drools piece — depends on #446 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#442 | Universal routing architecture design | L | High | Design-first; affects engine#439 |
| engine#448 | Worker(Plan.of(...)) function type | M | Med | Plan-based execution Phase 1 |

## Key References

- Design spec: `wksp/specs/2026-06-09-expression-evaluator-factory-design.md`
- PR #451: https://github.com/casehubio/engine/pull/451
- Epic #445 (Full Drools Integration): https://github.com/casehubio/engine/issues/445
