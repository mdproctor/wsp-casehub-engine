# Handoff — 2026-07-05

## What's Done

**#652 batch closed** — `c9aec072` on main. Five issues landed: #652 (CaseDefinition types/labels — `Set<Path>` classification + registry queries), #638 (MatchDegree on AgentCandidate via MatchedWorker), #608 (qhorus Store SPI import migration), #623 (fsitrading CI dispatch), #613 (soc — already done, closed).

Platform convention established: `types: Set<Path>` + `labels: Set<Path>` on every definable entity. Added to PLATFORM.md (`6b31ec7d` on parent). Deferred issues filed: work#291 (WorkItem/Template types), #653-#656 (persistence, vocabulary, instance-level).

Ledger import fix committed on main (`LedgerEntry` → `JpaLedgerEntry`, `LedgerEntryRepository` → `api.spi`). Refs casehubio/ledger#173.

Consumer repo commits for #644 still **not pushed** — devtown, aml, clinical, life, ops, soc, iot.

## Cross-Module

**Consumer repos need pushing** (local commits for #644 not yet on remote):
- devtown, aml, clinical, life, ops, soc, iot · XS · Low

**capabilityNames migration still open** (pre-existing):
- life#47, aml#85, devtown#117, desiredstate#50, parent#328 · S · Low

**desiredstate unblocked** by #652 — can now use `CaseDefinition.types` for response case classification.

## What's Left

- #646 — per-case CONTEXT_CHANGED serialization (Option B) · M · Med
- #649 — PlanItem multi-source-state CAS loops · S · Med
- #658 — fix YAML schema required list (apiVersion vs dsl, missing namespace/name) · XS · Low
- CaseMetaModelRepository + SubCaseGroupRepository naming cleanup — file issue · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
| #654 | Populate CaseMetaModel definition column during registration | S | Low | |
| #655 | Vocabulary validation for types/labels | M | Med | |
