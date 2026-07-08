# Handoff — 2026-07-08

## What's Done

**GoalBasedCompletion generalized (#582) landed on main** — GoalKind is now an interface with StandardGoalKind enum. GoalBasedCompletion<K> uses insertion-ordered map (first match wins). Goal.kind decoupled to String. YAML completion block is open with custom kind support. Design spec adversarially reviewed (3 rounds, 15 issues, $14.36). 7 implementation tasks, all modules green.

## What's Left

- #681 — code review findings from sweep (CBR cache bound, listener lifecycle, VocabularyRegistry instanceof) · S · Low
- #680 — thread tenancyId through event bus messages (deeper fix for #663) · M · Med
- #646 — per-case CONTEXT_CHANGED serialization · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo (14+ repos), scoped and documented |
| #648 | OutcomeRecorder.addAttestation | S | Med | Cross-repo (casehub-ledger SPI first) |
| #672 | Feature-level similarity in RetrievedExperience | S | Med | Cross-repo (neocortex API first) |
| #592 | External-backend recovery gap | M | Med | |
