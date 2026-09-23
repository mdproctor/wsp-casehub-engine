# HANDOFF — casehub-engine

## Last Session

Implemented #1148 (generalise evolution conductor) — 6 commits across the full 3-batch plan. Created 5 SPI interfaces (ImprovementCategoryProvider, ImprovementProposalSource, RegressionEvaluator, ConflictStrategy, DenyPatternProvider), 5 model records, 5 registries, CodeEvolutionCategoryProvider, EvolutionBootstrap, CodeEvolutionStages constants, CodeEvolutionMetadata utility, and 4 default domain implementations. Migrated ImprovementStage enum→String across 24 files. Refactored ImprovementGoalFormationStrategy into a domain-aware coordinator with 6-stage filtering pipeline. Cleaned ImprovementBudgetEnforcer (deny logic moved to CodeEvolutionDenyPatternProvider), deleted ConflictDetector (replaced by FilePathConflictStrategy). Added MultiDomainEvolutionTest. All api+runtime-core tests pass (512+). Full project build was running at session end — verify.

## Immediate Next Step

Use `work continue`. Queue has 3 remaining items (#1168, #1169, #1170) — all follow-ups from #1148 implementation, now in the .plan queue. Full project build passed clean. Start with `work next` to advance past #1148 to #1168, then execute sequentially (#1168 → #1169 → #1170). After all three, #1148 can be closed and `work end` run.

## Deferred Items (GitHub Issues)

- #1168 — Migrate HealthSnapshot→HealthScoreSnapshot (S / Low) — mechanical type migration, bridged by evaluator
- #1169 — Wire RegressionDetector to RegressionEvaluatorRegistry (S / Low) — depends on #1168
- #1170 — Remove targetRepo/targetPaths from ImprovementRequest (M / Low) — ~27 call sites

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
- Queue: slot `.plan` (position 6/9, #1148 active → advance to #1168)
- Epic: casehubio/engine#1139 (6/7 child issues closed)
