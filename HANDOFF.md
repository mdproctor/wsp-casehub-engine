# HANDOFF — casehub-engine

## Last Session

Completed the unified resolution pipeline (engine#1081). Batches 4 and 5 landed this session:

**Batch 4 — Retrieval Feedback:**
- `fireStepOutcomeObserver()` changed from `.get()` single-dispatch to multi-observer iteration
- DECLINED outcome mapping fixed in `handleSemanticFailure()` (`routingOutcome` derived per case)
- `RetrievalFeedbackObserver` — Layer 1 feedback via `StepOutcomeObserver`. Correlates dispatch-time experiences from EventLog metadata with worker outcomes (SUCCESS→RELEVANT, DECLINED→NOT_RELEVANT for declined agent, FAILURE→NOT_RELEVANT)
- `caseId` field added to `RetrievedExperience` (13th, backward-compatible) — `CbrRetrievalService.mapScoredCase()` threads `scored.caseId()` through

**Batch 5 — Candidate Presentation + Selection Feedback:**
- `publishJudgmentSchedule()` populates `_candidates.<bindingName>` with ranked summaries via `engineSet()` when experiences are non-empty
- `SelectionFeedbackRecorder` — Layer 3 feedback observing `PlanItemStateChangedEvent`. Validates `selectedCaseId` against presented candidates (rejects fabricated IDs). Selected→HIGHLY_RELEVANT, unselected above threshold→PARTIALLY_RELEVANT
- `casehub.cbr.outcome-weighting.enabled=true` default-on
- `cbr-playbook-guide.md` updated with Outcome Weighting and Retrieval Feedback sections

All 5 batches (foundation types, mixed retrieval, document ingestion, retrieval feedback, candidate presentation) are complete. Issue #1081 is ready for work-end.

## Pre-Existing Issues

- **CDI ambiguity in `@QuarkusTest` runtime tests:** `DefaultTestPrincipal` / `CurrentPrincipal` `AmbiguousResolutionException`. Pre-existing — affects `StepOutcomeObserverTest` and other `@QuarkusTest` classes.
- **YamlCaseHubTest case sensitivity:** 3 failures in api module (`"Minimal"` vs `"minimal"`). Pre-existing.

## References

- `specs/issue-1081-unified-resolution-pipeline/2026-09-11-unified-resolution-pipeline-design.md` — design spec
- `plans/2026-09-11-unified-resolution-pipeline.md` — implementation plan (all 5 batches complete)
- `blog/2026-09-11-mdp02-every-resolution-same-pipeline.md` — diary entry
