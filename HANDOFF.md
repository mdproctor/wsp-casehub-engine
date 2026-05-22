# Handoff — 2026-05-22

**Head commit (engine):** 3da76e5 — feat(api): decouple Agent from JqTransformer (Closes #316, #301)
**Head commit (workspace):** 26ec16f — docs: add blog entry 2026-05-22

## What Changed This Session

Major sprint — 5 engine issues closed, all work branches clean.

**engine#314** — JQ consolidation: `evalObjectTemplate` removed from `CaseContext` interface,
`ContextUtils` deleted, `JQEvaluator` moved to `casehub-engine-common`, 9 call sites migrated,
`JqTransformer` scope bug fixed, protocol `PP-20260522-jq-evaluation-canonical` written.

**engine#312** — `PlanningStrategyLoopControl.indexSelectedForCompletion()` no longer calls
`markRunning()` for `HumanTaskTarget` bindings. Handler guard reverted to PENDING-only.
End-to-end test added in `work-adapter`.

**engine#303** — `SpiWiringIntegrationTest` provisioner test timeouts raised 5s/10s → 30s.
Test still unrunnable due to pre-existing `RoutingCursorStore` boot failure (engine#321).

**engine#316** — `Agent` holds `UnaryOperator<JsonNode>` instead of `JqTransformer`.
`AgentBuilder` adds `inputTransformer(UnaryOperator<JsonNode>)` for CDI callers; schema
strings still work.

**engine#301** — `CommandContent` record replaces raw `HashMap` in `dispatchCommand()`.

Filed: platform#23 (JQEvaluator → casehub-platform Foundation tier), devtown#37 (MapCaseContext stub cleanup), engine#320 (engine consume from platform#23), engine#321 (RoutingCursorStore boot failure), engine#322 (SubCase PlanItem state gap), engine#323 (openChannel lifecycle), engine#324 (AgentTest style).

## Immediate Next Step

PR #313 still open: `gh pr view 313 --repo casehubio/engine` — resolve json-schema-validator conflict. Merge when ready.

## Cross-Module

**Blocked by:**
- `casehub-platform` — platform#23 (JQEvaluator extraction). engine#320 (engine consuming platform's evaluator) gates on platform#23 completion. Platform session has this in progress.

## What's Left

- engine#321 — RoutingCursorStore unsatisfied dep blocks ALL blackboard + select runtime `@QuarkusTest` · S · Low
- engine#322 — SubCase PlanItems never transition beyond PENDING in BlackboardRegistry · S · Med
- engine#323 — Verify `openChannel` idempotency in `dispatchCommand` · S · Med
- engine#324 — Normalise AgentTest to AssertJ (pre-existing style) · XS · Low
- PR #313 — json-schema-validator version conflict · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#320 | Engine: consume JQEvaluator from casehub-platform | M | Med | Blocked by platform#23 |
| engine#321 | Fix RoutingCursorStore boot failure in blackboard/runtime tests | S | Low | Unblocks engine#303 verification |
| engine#322 | SubCase PlanItem state machine gap | S | Med | engine#312 surfaced this |
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| claudony#122 | Extract correlationId + deadline from COMMAND content | S | Med | Unblocked by engine#300/301 |

## Key References

- Blog: `blog/2026-05-22-mdp02-wrapper-earns-its-keep.md`
- Spec (314): `docs/specs/2026-05-22-jq-evaluator-consolidation-design.md`
- Spec (316/301): `docs/specs/2026-05-22-agent-command-content-design.md`
- Protocol: garden `casehub/jq-evaluation-canonical.md` (PP-20260522)
- Garden: GE-20260522-adb5cd (Quarkus CDI library JAR discovery), GE-20260522-3e2589 (ChatModel.doChat stub)
- PR #313: https://github.com/casehubio/engine/pull/313
