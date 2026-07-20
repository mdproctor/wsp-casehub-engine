# Handoff — 2026-07-20

## What's Done

- **engine#730**: Case queue — delivered, merged to main, pushed to upstream
  - New `casehub-engine-queue` module (optional, classpath-activated)
  - CaseInstance.labels + CaseDefinition.labelRules with YAML/Java DSL
  - CaseLabelEvaluator, CaseQueueEntryManager, CaseQueueService, CaseLabelReconciler
  - Design-reviewed (4 rounds, 23 issues, all resolved, $21)
  - 48 tests across 8 test classes
- **engine#761**: Wired CbrRetrievalService into case startup — inject experiences into CaseContext (on main)

## Cross-Module

**Blocked by:**
- `platform` — generic labelling infrastructure (platform#187) — CLOSED, no longer blocking

## Immediate Next Step

- Pick up follow-on issues from #741: engine#754 (HumanTask CBR impl), #755 (constraint impl), #756 (work repo consumption), #757 (group scoring)
- Or pick up engine backlog items

## Session Context

- PR casehubio/engine#753 CI still failing (build conclusion: FAILURE) — may need investigation
- engine#730 branch `issue-730-case-queue` is stamped closed
- Four follow-on issues from #741: engine#754-757
