# Handoff — 2026-07-06

## What's Done

**#649, #662 closed** — `17e307f3` on main. PlanItem CAS loops (markCompleted, markFaulted, markObsolete, markCancelled) close the TOCTOU gap for concurrent terminal transitions. Handler ISE catching added to WorkerRetryExhaustionHandler and PlanItemFaultHandler. Full dual-stack for CaseMetaModelRepository and SubCaseGroupRepository — reactive interfaces renamed to Reactive* prefix, blocking counterparts created, in-memory and JPA implementations split following established pattern. MemorySubCaseGroupRepository renamed to InMemorySubCaseGroupRepository.

## Cross-Module

**desiredstate unblocked** by #652 (prior session) — can use `CaseDefinition.types` for response case classification.

**devtown needs follow-up** — #667: two devtown classes extend renamed engine implementations (`DevtownReactiveCaseMetaModelRepository`, `DevtownReactiveSubCaseGroupRepository`). IntelliJ auto-renamed them but the devtown repo needs a commit.

## What's Left

- #646 — per-case CONTEXT_CHANGED serialization (Option B) · M · Med
- #666 — consolidate WorkerRetryExhaustionHandler + PlanItemFaultHandler (both subscribe to WORKER_RETRIES_EXHAUSTED) · S · Med
- #669 — SubCaseMofNOutputMappingTest suite interaction issue · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
| #654 | Populate CaseMetaModel definition column during registration | S | Low | |
| #655 | Vocabulary validation for types/labels | M | Med | |
| #667 | Devtown cross-repo rename propagation | S | Low | Unblocked by this session |
