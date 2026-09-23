# HANDOFF — casehub-engine

## Last Session

Brainstorming and planning for #1148 (generalise evolution conductor). Advanced queue past #1146 (already closed) to #1148 (position 5/6). Completed full design cycle: 8 decisions captured, spec written and self-reviewed (5 fixes), post-spec standard review (3 rounds, 15 issues, 14 verified, 1 accepted, APPROVED). Review expanded scope from 3 to 5 SPIs — added ConflictStrategy and DenyPatternProvider for filtering-pipeline generalisation. Implementation plan written (6 tasks, 3 batches), self-reviewed (1 gap fixed — ImprovementRequest field removal step added).

## Immediate Next Step

Execute the implementation plan at `plans/2026-09-23-generalise-evolution-conductor.md`. Use executing-plans skill. Start with Batch 1 Task 1 (create API model types and SPI interfaces). This is L/XL scale work — expect 2-3 sessions across the 3 batches.

The spec at `specs/issue-1148-generalise-evolution-conductor/2026-09-23-generalise-evolution-conductor-design.md` is the design authority. The plan sequences the spec into TDD tasks.

## Key Design Decisions (D1-D8)

- D1: CapabilityArea IS HealthSensor — no new SPI needed
- D2: ImprovementCategoryProvider SPI (multi-instance, contributes categories + stages)
- D3: ImprovementProposalSource SPI (multi-instance, generates ImprovementRequests directly)
- D4: RegressionEvaluator SPI (pluggable evaluators, RegressionDetector stays as orchestrator)
- D5: Code-evolution stays in runtime-core as defaults
- D6: Worker infrastructure IS the executor — no new SPI
- D7: Direct proposals bypass signal-consensus (consensus becomes one source's implementation detail)
- D8: ImprovementStage enum → string (domains define stage sequences via provider)

## References

- Spec: `specs/issue-1148-generalise-evolution-conductor/2026-09-23-generalise-evolution-conductor-design.md`
- Decisions: `specs/issue-1148-generalise-evolution-conductor/decisions.md`
- Plan: `plans/2026-09-23-generalise-evolution-conductor.md`
- Queue: slot `.plan` (position 5/6, #1148 active)
- Epic: casehubio/engine#1139 (6/7 child issues closed)
