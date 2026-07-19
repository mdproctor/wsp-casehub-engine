# Handoff — 2026-07-19

## What's Done

- **engine#741**: HumanTaskRoutingStrategy SPI — CBR routing enrichment for humanTask bindings. Delivered and merged.
  - New SPI: `HumanTaskRoutingStrategy`, `HumanTaskRoutingContext`, `HumanTaskCandidates`, `HumanTaskRoutingResult` (sealed: Enriched | Unchanged | Escalated)
  - `ExperienceAnalyser` generalised with `Predicate<ExperiencePlanStep>` overload
  - `CbrCaseRetainObserver` includes humanTask PlanItems in plan traces (capabilityName nullable)
  - Handler plumbing threads experiences through humanTask dispatch, calls strategy
  - Cross-repo: neocortex `PlanTrace`/`AdaptedStep` capabilityName nullable (companion commit)
  - Design-reviewed (4 rounds, 15 issues, all resolved)
  - PR: casehubio/engine#753 (updated — now covers #741 + #752)

## Deferred Items

- `ExperienceAnalyser`: handle SUBSTITUTED step attribution properly when `originalWorkerName` is available (wsp-casehub-engine#1)
- engine#730 (case queue implementation) blocked by platform#175

## Immediate Next Step

- Merge PR casehubio/engine#753 when CI passes (covers both #741 and #752)
- Push neocortex companion commit to upstream

## Session Context

- neocortex 0.2-SNAPSHOT with nullable capabilityName must be installed in local maven for engine to compile
- Four follow-on issues filed: engine#754 (CBR impl), #755 (constraint impl), #756 (work repo consumption), #757 (group scoring)
