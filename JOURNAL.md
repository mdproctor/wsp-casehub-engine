# Design Journal — issue-1180-evolution-conductor-ui-in-devtown

## 2026-10-07 — Session 1: Design + Batch 1 Implementation

**Branch:** `issue-1180-evolution-conductor-ui-in-devtown` (engine, devtown, blocks-ui)
**Issue:** casehubio/engine#1180, casehubio/devtown#230

### What happened

Brainstormed and designed the full evolution conductor integration for devtown as first consumer. Captured 7 design decisions (all quick picks following established patterns), wrote and self-reviewed the design spec, then created the implementation plan (4 batches, 10 tasks).

Implemented Batch 1: 5 devtown capability areas (CiReliability, ReviewQuality, MergeQueueHealth, ReviewerTrust, SlaCompliance) + DevtownCategoryProvider (5 categories, 5-stage pipeline). All 22 tests green.

### Key decisions

- D1: Devtown areas register alongside engine's 10 default areas via CDI
- D2: Core 5 areas covering the PR pipeline lifecycle
- D3: DevtownEvolutionApi facade enriches views at the server layer
- D4: 9th dashboard tab via hostPanel
- D5: 5 review-pipeline improvement categories with shorter 5-stage pipeline
- D6: Deep links via enriched strings + custom TabDefinition extensions (no blocks-ui changes)
- D7: Singleton evolution case auto-bootstrapped at startup

### Adjustment from plan

Capability areas placed in `app` module (not `domain`) because domain lacks engine dependencies. Package: `io.casehub.devtown.app.evolution`.

### What's next

Batch 2: Filtering SPIs (DevtownConflictStrategy, DevtownDenyPatternProvider, DevtownRegressionEvaluator, DevtownProposalSource)
Batch 3: API facade + REST resource + case bootstrap
Batch 4: Frontend Evolution tab integration
