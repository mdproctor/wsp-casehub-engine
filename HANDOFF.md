# Handoff — 2026-07-06

## What's Done

**#658, #464, #659 closed** — `bdfb8675` on main. Three commits landed: #658 (YAML schema required list fix — apiVersion→dsl, add namespace/name), #464 (panel→layer rename across 48 files on CaseContext API), #659 (flaky test root causes — PlanItem re-creation, handler ordering, settlement race). Also fixed pre-existing test issues: stale ScoreType import, missing reactive repo alternatives in memory profile.

## Cross-Module

**desiredstate unblocked** by #652 — can now use `CaseDefinition.types` for response case classification.

## What's Left

- #646 — per-case CONTEXT_CHANGED serialization (Option B) · M · Med
- #649 — PlanItem multi-source-state CAS loops · S · Med
- CaseMetaModelRepository + SubCaseGroupRepository naming cleanup — file issue · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
| #654 | Populate CaseMetaModel definition column during registration | S | Low | |
| #655 | Vocabulary validation for types/labels | M | Med | |
