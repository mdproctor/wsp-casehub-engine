# HANDOFF — casehub-engine

## Last Session

Implemented Batch 4 of the unified resolution pipeline (engine#1081). Three changes:

1. **Handler fixes in WorkflowExecutionCompletedHandler:** `fireStepOutcomeObserver()` changed from `.get()` single-dispatch to iterating all `Instance<StepOutcomeObserver>` beans (matching the `CaseOutcomeObserver` pattern). DECLINED outcome mapping fixed — `handleSemanticFailure()` now derives `routingOutcome` per switch case (DECLINED→`RoutingOutcome.DECLINED`, FAILED/EXPIRED→`RoutingOutcome.FAILURE`) instead of hardcoded `FAILURE` for all. Existing test updated to assert `DECLINED`.

2. **RetrievalFeedbackObserver:** New `@ApplicationScoped StepOutcomeObserver` in `runtime/internal/routing/`. Correlates CBR retrieval traces with worker outcomes: reads experiences from `WORKER_SCHEDULED` EventLog metadata, maps outcome to `CbrFeedbackOutcome` (SUCCESS→RELEVANT, DECLINED→NOT_RELEVANT for declined agent's experiences, FAILURE→NOT_RELEVANT, gate/cancel/obsolete→no signal), records via `CbrRetrievalTracker.feedback()`. Transparent no-op when tracker absent via `Instance<CbrRetrievalTracker>.isResolvable()`. 9 unit tests with Mockito.

3. **caseId on RetrievedExperience:** Added as 13th field (`@Nullable String caseId`) with backward-compatible 12-arg constructor. `CbrRetrievalService.mapScoredCase()` now threads `scored.caseId()` through to `RetrievedExperience` for both `ResolvedCase` and `ResolutionGuide` paths. Required by `RetrievalFeedbackObserver` to identify which CBR case entry each feedback signal targets.

## Immediate Next Step

Batch 5: candidate presentation via JudgmentTarget (Task 6) + selection feedback Layer 3 (Task 7). Task 6 is the most complex — enriches judgment dispatch with ranked CBR candidates written to `_candidates.<bindingName>` context path, with `ResolutionSelection` validation in the judgment completion path.

## Cross-Module

- **neocortex (casehubio/neocortex)** — #320 and #321 delivered and installed. No further blockers.

## Pre-Existing Issues

- **CDI ambiguity in `@QuarkusTest` runtime tests:** `DefaultTestPrincipal` / `CurrentPrincipal` `AmbiguousResolutionException`. Pre-existing — affects `StepOutcomeObserverTest` and other `@QuarkusTest` classes in the runtime module. Not caused by this branch's changes.
- **YamlCaseHubTest case sensitivity:** 3 failures in api module (`"Minimal"` vs `"minimal"`). Pre-existing.

## References

- `specs/issue-1081-unified-resolution-pipeline/2026-09-11-unified-resolution-pipeline-design.md` — reviewed design spec
- `plans/2026-09-11-unified-resolution-pipeline.md` — implementation plan (Batches 1-4 complete, 5 remaining)
- `blog/2026-09-11-mdp02-every-resolution-same-pipeline.md` — diary entry
