# Gap Analysis — Issue #1115 Continuous Evolution Loop

**Date:** 2026-10-26
**Existing spec:** `docs/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md`
**Method:** Systematic comparison of spec against production code via 4 parallel investigations

## Executive Summary

The design spec is comprehensive and sound. All major components exist as classes with
passing unit tests. **The fundamental problem is that the production CDI wiring is
disconnected — the loop is open.** Components work in isolation but are not connected
in the runtime graph.

---

## CRITICAL — Loop Not Closed

### C1. EvolutionTicker is dead code

`EvolutionTicker` is produced as a CDI bean (RuntimeBeans.java:1360-1381) but **never
consumed**. No production code calls `tick()`.

`CaseContextChangedEventHandler.convergenceDetection()` calls
`improvementStrategy.proposeImprovements()` **directly** (line 1498-1513), bypassing
the entire gate pipeline (evolution-enabled, health refresh, regression monitoring,
circuit breaker evaluation).

**Impact:** All safety gates are bypassed in the real event-driven path.

### C2. No timer backstop

No `@Scheduled`, `ScheduledExecutorService`, or periodic trigger invokes the ticker.
The timer backstop described in the spec does not exist.

### C3. ImprovementOutcomeEventCapture is not a CDI bean

The class writes to all 6 layers internally (EventLog, signals, CBR, budget, category
tracker, regression detector) but is never instantiated in production. Nobody fires
`ImprovementCaseCompleted` events.

**Impact:** Outcome feedback never flows. Category suppression data never arrives.
Regression detection never starts monitoring. Anti-oscillation history never accumulates.
The loop is open at the outcome boundary.

---

## HIGH — Safety Gaps

### H1. Rollback case spawning not implemented

`RegressionDetector.onMetricsDegraded()` records in RollbackHistory and pauses the
category, but **never spawns a rollback case** from the `self-improvement-rollback`
template. The template exists with correct bindings. The `ImprovementRevertWorker`
exists. But no code instantiates the template.

### H2. Circuit breaker restart recovery not called

`ImprovementCircuitBreaker.restoreFromEventLog()` is fully implemented. It queries
EventLog for the latest circuit breaker event and reconstructs state. But no startup
hook calls it. After restart, circuit breaker always starts CLOSED regardless of
previous state.

### H3. RegressionDetector missing from structural deny list

`CodeEvolutionDenyPatternProvider.STRUCTURAL_DENIED_PATTERNS` includes most safety
components but omits `RegressionDetector`. A self-improvement could modify the
regression detector — a safety violation.

---

## MEDIUM — Missing Features

### M1. CBR-informed goal formation not wired

`proposeImprovements()` does not consult CBR for similar past outcomes.
`ImprovementCbrProjector.project()` is a logging stub that writes a debug message
but does not store to CBR.

### M2. Anti-cascade guard not implemented

Before rollback, the spec says the detector should check for later improvements that
depend on the regressed change. No dependency tracking between improvements exists.

### M3. ProposalFilteringSummary never populated

`TickTrace.TickOutcome.ProposalGenerated` always passes `null` for filtering summary.
`ImprovementGoalFormationStrategy.proposeImprovements()` does not build or return
filtering statistics. The record type exists but is never used.

### M4. Three YAML codegen entries missing

`RollbackPolicy`, `HealthPolicy`, and `ResearchMethodology` have no entries in
`yaml-record-mappings.yaml`. They cannot be deserialized from YAML case definitions.

### M5. No DevTown improvement workers

DevTown proposes improvements via `DevtownProposalSource` (5 categories from 5
capability areas) but has no workers to execute the "apply" stage. The loop is open
at the execution boundary.

---

## LOW — Test Gaps

### L1. Three integration tests missing

- `RegressionRollbackIntegrationTest` — rollback workflow not tested end-to-end
- `CircuitBreakerIntegrationTest` — dedicated circuit breaker lifecycle not tested
- `ConflictSerializationIntegrationTest` — conflict serialization round-trip not tested

Existing `ContinuousEvolutionIntegrationTest` (7 tests) partially covers these
scenarios but does not substitute for dedicated tests.

### L2. ResearchPipelineOrchestratorTest is thin

Single test method (`pipelineExecutesAllSteps`).

---

## What's Complete

| Component | Status |
|-----------|--------|
| ImprovementConfig (13 fields) | All present |
| EvolutionTicker gate pipeline | Matches spec + extra deny filter |
| proposeImprovements() filter pipeline | 6 stages correctly wired |
| TickTrace / TickTraceBuffer | Fully implemented |
| Case templates (self-improvement, self-improvement-rollback) | Both exist |
| EvolutionBootstrap (5 SPI registrations) | Complete |
| HealthScoreTracker.computeScore() | Real aggregation from 10 CapabilityAreas |
| Backwards-compatible constructors | 3-tier (5, 11, 13 params) |
| DevTown: proposal source, 5 capability areas, conflict strategy, regression evaluator, category provider, REST API, UI packages, bootstrap | All present |
| Unit tests | 13 of 16 spec tests exist and pass |
