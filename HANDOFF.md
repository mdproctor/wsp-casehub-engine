*Updated: #651, blocks#30 closed — removed from backlog.*

# Handoff — 2026-07-05

## What's Done

Six issues landed on main (`69c90c67`): #651 (AgentRoutingContext tenancyId), #650 (mandatory rationale), #640 (dual-stack blocking/reactive repos), #626 (CaseEventRecorder + orchestration events), #644 (7 consumer repos migrated), #583 (CI dispatch).

Consumer repo commits on their respective `main` branches but **not pushed** — devtown, aml, clinical, life, ops, soc, iot all have local-only commits for #644.

quarkmind#226 filed — deferred migration until quarkmind returns to main.

## Cross-Module

**Consumer repos need pushing** (local commits for #644 not yet on remote):
- devtown, aml, clinical, life, ops, soc, iot · XS · Low

**capabilityNames migration still open** (pre-existing, not addressed this batch):
- life#47, aml#85, devtown#117, desiredstate#50, parent#328 · S · Low

## What's Left

- #646 — per-case CONTEXT_CHANGED serialization (Option B) · M · Med
- #649 — PlanItem multi-source-state CAS loops · S · Med
- CaseMetaModelRepository + SubCaseGroupRepository naming cleanup — file issue · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
