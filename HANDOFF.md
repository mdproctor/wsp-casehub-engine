# HANDOFF — casehub-engine

## Last Session

<<<<<<< HEAD
Completed 3 issues (#1162, #1163, #1164) advancing the persistence coherence epic from position 12/17 to 15/17. Removed vestigial duplicate JpaPlanItemStore from work-adapter (reactive Panache era relic), created abstract contract tests for 5 SPIs (SubCaseGroupRepository, CrossTenant*, PlanVersionStore, ExecutionSnapshotStore), and converted JPA tests to extend them. Fixed two missing Flyway migrations (context_snapshot, case_queue_entry) that were blocking all @QuarkusTest tests — persistence-hibernate now runs 34/34 green.

## Immediate Next Step

`work next` to advance to #1165 — integration test for full DLQ replay path. Topic shift from persistence coherence into resilience/DLQ.

## References

| Artifact | Path |
|----------|------|
| Design spec | `wksp/specs/issue-1150-persistence-coherence/2026-09-23-case-context-recovery-strategy-design.md` |
| Decisions | `wksp/specs/issue-1150-persistence-coherence/decisions.md` |
=======
Completed the final 3 issues in the queue (#1168, #1169, #1170) — all follow-ups from #1148 generalisation. Closed #1148 and all child issues on GitHub. 513 runtime-core tests pass throughout.

### #1168 — Migrate HealthSnapshot to HealthScoreSnapshot (5 files)
Deleted `HealthScoreTracker.HealthSnapshot` inner record. All usages now use the API record `HealthScoreSnapshot` directly. Removed bridge conversion in `HealthScoreDeltaRegressionEvaluator`.

### #1169 — Wire RegressionDetector to RegressionEvaluatorRegistry (5 files)
Replaced direct `ConfidenceScorer` dependency with domain-filtered evaluation through `RegressionEvaluatorRegistry`. Detector resolves domain via `ImprovementCategoryRegistry.domainForCategory()`, filters evaluators by `domainId()`, takes max confidence across matches.

### #1170 — Remove targetRepo/targetPaths from ImprovementRequest (19 files)
Deleted vestigial `targetRepo` and `targetPaths` fields and 7-arg backward-compatible constructor. Workers now read paths exclusively via `CodeEvolutionMetadata.extractPaths()`. Updated 41 constructor call sites across 12 test files.

## Immediate Next Step

Queue has 3 follow-up items. Use `work continue` → `work next` to advance to #1143.

## What's Next

| # | Title | Scale | Complexity | Notes |
|---|---|---|---|---|
| #1143 | Research pipeline checkpoint | S | Med | Shows OPEN on GitHub — verify whether code landed or needs implementation |
| #1180 | Surface evolution conductor UI in devtown | M | Med | blocks-ui composables + devtown as first consumer. Design direction captured in issue |
| #1181 | Soredium as agent workflow methodology | L | High | Experimental. Conductor triggers soredium-managed Claude Code sessions. One LLM types in another's terminal |

## Key Design Decisions (This Session)

- **D9: blocks-ui/devtown split** — blocks-ui gets composable primitives (health panel, gate card, improvement stream, conductor inbox, tick timeline, workbench shell). Devtown gets a page that instantiates the workbench with code-evolution sensors and developer-specific chrome. Split principle: blocks-ui = what to show, devtown = where to show it.
- **D10: Devtown as forcing function** — devtown is the first consumer of the evolution loop applied to its own development pipeline (design → code → PRs → builds). Components built for observing the dev pipeline are naturally reusable for trading, AML, clinical, SOC.
- **D11: Soredium as agent methodology** — the conductor provides what/when, soredium provides how. Executing agents are standard Claude Code sessions with full skill discipline. The garden's retain step provides cross-session learning.

## References

- Spec: `specs/issue-1148-generalise-evolution-conductor/2026-09-23-generalise-evolution-conductor-design.md`
- Decisions: `specs/issue-1148-generalise-evolution-conductor/decisions.md`
- Plan: `plans/2026-09-23-generalise-evolution-conductor.md`
- Epic: casehubio/engine#1139 (CLOSED — all child issues done)
- Next epic: casehubio/engine#1149 (production readiness, UI, blog)
>>>>>>> issue-1141-cdi-event-wiring
