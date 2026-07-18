# Handoff — 2026-07-18

## What's Done

- **engine#752**: Adaptation-aware CBR scoring and prompt rendering — delivered.
  - `ExperienceAnalyser.workerSuccessRates()` now skips SUBSTITUTED steps alongside ADDED
  - `CbrRoutingPromptSection` annotates non-RETAINED adapted steps (`[ACTION: reason]`), excludes ADDED/SUBSTITUTED from outcome aggregates
  - `CbrAgentRoutingStrategy.analyseExperiences()` consolidated to delegate to `ExperienceAnalyser` — eliminates 35 lines of duplicated scoring logic
  - Companion commit in casehub-blocks (`e37e638`)
  - PR: casehubio/engine#753 (open)

## Deferred Items

- `ExperienceAnalyser`: handle SUBSTITUTED step attribution properly when `originalWorkerName` is available — currently skipped entirely (wsp-casehub-engine#1, still relevant)
- casehub-life can remove manual `PlanAdapter` call from `LifeCaseService.startCase()` — engine now handles adaptation automatically
- engine#730 (case queue implementation) blocked by platform#175 (generic queue toolkit)

## Immediate Next Step

- Merge PR casehubio/engine#753 when CI passes
- Push blocks commit (`e37e638`) to upstream if blocks has a fork model

## Session Context

- neocortex 0.2-SNAPSHOT with #161 must be installed in local maven for engine to compile
