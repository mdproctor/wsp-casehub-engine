*Updated: #667, #678 closed — removed from backlog.*

# Handoff — 2026-07-07

## What's Done

**S/XS backlog sweep (#678) landed on main** — 12 issues closed in one branch. Main divergence from previous session reconciled (cherry-picked 9 commits onto origin/main, rescued `ab60f7b8` #18 submit overload from orphaned branch). #669 flaky test root-caused and fixed (non-deterministic event bus delivery order — garden entry GE-20260707-ee0718). CI green.

Key new API surface: `ExecutionOrigin`, `RetryState`, `CaseContext.onChange()`/`onAnyChange()`, `Binding.producedKeys`, `CbrRetrievalTiming`, QhorusMessageSignalBridge STATUS routing. CLAUDE.md synced. Design spec adversarially reviewed (3 rounds, 16 issues, all resolved).

## Cross-Module

**devtown needs follow-up** — #667: two devtown classes extend renamed engine implementations.

## What's Left

- #681 — code review findings from sweep (CBR cache bound, listener lifecycle, VocabularyRegistry instanceof) · S · Low
- #680 — thread tenancyId through event bus messages (deeper fix for #663) · M · Med
- #646 — per-case CONTEXT_CHANGED serialization · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo (14+ repos), scoped and documented |
| #648 | OutcomeRecorder.addAttestation | S | Med | Cross-repo (casehub-ledger SPI first), scoped |
| #672 | Feature-level similarity in RetrievedExperience | S | Med | Cross-repo (neocortex API first), scoped |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
