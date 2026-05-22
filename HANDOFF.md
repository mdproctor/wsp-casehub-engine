# Handoff — 2026-05-22

**Head commit (engine):** a9fd4ec — docs(issue-320): apply design journal to DESIGN.md
**Head commit (workspace):** 17b8845 — docs: add blog entry 2026-05-22 — Clearing the Interim Address

## What Changed This Session

Closed engine#320: JQEvaluator, ValidationResult, SecretManager, ConfigManager and exception types removed from `casehub-engine-common`; all 40 call sites now import from `casehub-platform-expression` / `casehub-platform-api`. Four test `application.properties` updated with `quarkus.index-dependency.platform-expression`. `ValidationResultTest` updated to match platform's stricter null contract. PR #328 open on casehubio/engine.

Closed stale PR #313 (json-schema-validator pin was already on main via `953f725`).

Fixed publish-blog skill: removed hardcoded `docs/_posts/` path (now resolves from CLAUDE.md routing), eliminated `blog_router.py` dependency (logic applied inline from YAML config). `blog_router.py` and its 37-test suite deleted from cc-praxis as dead code. PP-20260522-bbd139 (`platform-cdi-index-dependency`) captured.

## Immediate Next Step

Merge PR #328 once CI passes, then start engine#321 — fix RoutingCursorStore unsatisfied dep (blocks ALL blackboard + runtime `@QuarkusTest`).

## What's Left

- PR #328 — open, CI running · XS · Low
- engine#321 — RoutingCursorStore boot failure blocks all blackboard/runtime @QuarkusTest · S · Low
- engine#322 — SubCase PlanItems never transition beyond PENDING in BlackboardRegistry · S · Med
- engine#323 — Verify `openChannel` idempotency in `dispatchCommand` · S · Med
- engine#324 — Normalise AgentTest to AssertJ style · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#321 | Fix RoutingCursorStore boot failure | S | Low | Unblocks engine#303 verification |
| engine#322 | SubCase PlanItem state machine gap | S | Med | engine#312 surfaced this |
| claudony#122 | Extract correlationId + deadline from COMMAND | S | Med | Unblocked by engine#300/301 |
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |

## Key References

- Blog: `blog/2026-05-22-mdp03-clearing-interim-address.md`
- PR #328: https://github.com/casehubio/engine/pull/328
- Protocol PP-20260522-bbd139: garden `casehub/platform-cdi-index-dependency.md`
- cc-praxis fix: `issue-96-diary-to-log-sweep` branch (publish-blog + blog_router.py removal)
