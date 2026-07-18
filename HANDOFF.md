# Handoff — 2026-07-18

## What's Done

- **engine#738**: Wire PlanAdapter into CbrRetrievalService pipeline — delivered.
  - `ExperiencePlanStep` enriched with `adaptationAction` and `adaptationReason` (nullable Strings, engine-api owned)
  - `ExperienceAnalyser.workerSuccessRates()` filters ADDED steps (adapter recommendations excluded from statistics)
  - `CbrRetrievalService` injects `PlanAdapter` (blocking SPI, same pattern as `CbrCaseMemoryStore`)
  - For `PlanCbrCase` results, calls `adapt()` inside `mapScoredCase()` — REMOVED steps filtered, adapter failure falls back to raw mapping
  - Depends on neocortex#161 (caseType parameter on PlanAdapter — delivered separately)
  - Design spec: `docs/specs/2026-07-18-planadapter-cbr-wiring-design.md` (5-round adversarial review, $12.59)
  - Squashed to single commit `6752ab1b`, pushed to upstream/main

## Deferred Items

- `ExperienceAnalyser`: handle SUBSTITUTED steps — outcome misattributed to substituted worker (wsp-casehub-engine#1)
- `CbrRoutingPromptSection`: annotate adapted steps in LLM routing prompt (wsp-casehub-engine#2)
- Downstream consumers (`TrustWeightedAgentStrategy`, `CbrAgentRoutingStrategy`) don't yet interpret adaptation fields

## Immediate Next Step

- casehub-life can remove manual `PlanAdapter` call from `LifeCaseService.startCase()` — the engine now handles it automatically
- engine#730 (case queue implementation) is blocked by platform#175 (generic queue toolkit)

## Session Context

- neocortex 0.2-SNAPSHOT with #161 must be installed in local maven for engine to compile
- Pre-existing `eraseByScope` compilation errors in recording test stubs fixed as part of this branch
