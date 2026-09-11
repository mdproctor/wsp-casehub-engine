# HANDOFF — casehub-engine

## Last Session

Designed and specced the unified resolution pipeline (engine#1081). 12 design decisions captured, spec went through 3-round standard review (24 issues, 19 fixed). Implemented Batch 1 (foundation types): CBR naming cleanup across 12 files, new API types (ResolutionSourceType, DocumentStep, ResolutionSelection), RetrievedExperience extended with 3 new fields. All 63 tests green.

## Immediate Next Step

Land neocortex prerequisites before continuing Batch 2: (1) GuidanceStep record + features field + withFeatures()/withSteps() on ResolutionGuide in neocortex memory-api, (2) feedback() method on CbrRetrievalTracker in neocortex memory-api. File issues in casehubio/neocortex.

## Cross-Module

- **neocortex (casehubio/neocortex)** — two changes needed before engine Batches 2-5 can proceed: GuidanceStep on ResolutionGuide, CbrRetrievalTracker.feedback(). No issues filed yet — create them first.

## References

- `specs/issue-1081-unified-resolution-pipeline/2026-09-11-unified-resolution-pipeline-design.md` — reviewed design spec
- `specs/issue-1081-unified-resolution-pipeline/decisions.md` — 12 design decisions
- `plans/2026-09-11-unified-resolution-pipeline.md` — implementation plan (7 tasks, 5 batches)
- `blog/2026-09-11-mdp02-every-resolution-same-pipeline.md` — diary entry
