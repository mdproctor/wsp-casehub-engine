# Handoff — casehub-engine

## Last Session

Fixed workspace corruption (wksp symlink → non-git `work-work` dir) with TDD'd S9 corruption
check in soredium (commit `c9a4de5`). Repaired all 23 engine slot wksp symlinks. Then designed
and began implementing #943 (per-expression override for data transform projections). Core type
migration complete: `CapabilityTarget`, `Binding`, `WorkerScheduleEvent`, `DefaultPersistentScope`
all carry `ExpressionEvaluator` and route through `ExpressionEngineRegistry.transform()`.
4 of 8 plan tasks done.

## Immediate Next Step

Run `/work` to continue. Tasks 3-5, 8 remain: `SubCaseMapping.Expression` type change (Task 3),
YAML mapper `resolveExpression()` wiring for projection fields (Task 4), `AgentConverter`
registry pass-through (Task 5), dead `evalJq` method cleanup (Task 8). Plan at
`plans/2026-08-21-per-expression-transform-override.md`, spec at
`specs/issue-943-per-expression-transform-override/`.

## References

- Spec: `specs/issue-943-per-expression-transform-override/2026-08-21-per-expression-transform-override-design.md`
- Plan: `plans/2026-08-21-per-expression-transform-override.md`
- Soredium fix: commit `c9a4de5` — S9 orphaned wksp check + workspace_ok cross-check
