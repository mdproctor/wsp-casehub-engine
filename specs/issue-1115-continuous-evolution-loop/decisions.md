# Decisions — Issue #1115 Gap Closure

## D1: Scope — full gap closure

**Choice:** Close all gaps (C1-C3, H1-H3, M1-M5, L1-L2) in a single pass
**Alternatives:**
- Critical only (C1-C3) — leaves safety gaps open
- Critical + High — defers DevTown workers and medium items
**Rationale:** The components are already built and tested in isolation. The work is
wiring, not greenfield design. Batching across sessions is acceptable.
**Trade-offs:** Large scope; may span multiple sessions
**Sources:** gap-analysis.md
**Exploration:** quick
**Status:** captured

## D2: Rollback case spawning mechanism

**Choice:** CDI event pattern — `RegressionDetector` fires `RollbackRequested` event,
a separate `RollbackCaseSpawner` handler creates the case via `GoalFormationService`
**Alternatives:**
- Direct case creation from RegressionDetector — simpler but couples detection to lifecycle
- Queue-based — adds infrastructure for no clear benefit
**Rationale:** Separation of concerns. RegressionDetector focuses on detection and
confidence scoring. Case creation is a separate responsibility that needs
GoalFormationService access. The event pattern follows the existing CaseHub CDI event
conventions.
**Trade-offs:** Extra class (RollbackCaseSpawner) and event type (RollbackRequested)
**Sources:** RegressionDetector.java, self-improvement-rollback.yaml, GoalFormationService
**Exploration:** quick
**Status:** captured

## D3: CBR integration scope

**Choice:** Make `ImprovementCbrProjector` write to the CBR `RetrieverService` when
available (optional `Instance<>` injection). Add optional CBR read in
`proposeImprovements()` for similar past failures. Graceful no-op when CBR unavailable.
**Alternatives:**
- Skip CBR entirely — simpler but loses learning capability
- Require CBR — breaks engine-only mode
**Rationale:** The engine-only fallback (RollbackHistory + CategoryTracker) is
sufficient for safety. CBR adds richer pattern matching but must not be required.
Optional injection with graceful fallback matches the existing SPI pattern.
**Trade-offs:** CBR learning only active when neocortex is deployed
**Sources:** ImprovementCbrProjector.java, ImprovementGoalFormationStrategy.java,
2026-09-20 spec §2 (CBR-informed goal formation)
**Exploration:** quick
**Status:** captured

## D4: Anti-cascade guard mechanism

**Choice:** Before rollback spawning, query `BudgetEnforcer.activeImprovementRequests()`
for path overlap with the regressed improvement's targets. If overlap found, downgrade
to signal + pause instead of auto-revert.
**Alternatives:**
- Full dependency graph between improvements — over-engineered for current needs
- Skip anti-cascade — risks cascading failures
**Rationale:** Reuses existing `FilePathConflictStrategy` path-overlap logic. Simple,
conservative, and consistent with the conflict detection approach already in the
filter pipeline. False positives (unnecessary downgrades) are safe.
**Trade-offs:** Coarse-grained — same-directory overlap may block valid rollbacks
**Sources:** FilePathConflictStrategy.java, BudgetEnforcer.activeImprovementRequests()
**Exploration:** quick
**Status:** captured

## D5: DevTown improvement workers — direct service calls

**Choice:** Workers are CDI beans in the devtown repo that call internal DevTown
services to apply operational changes
**Alternatives:**
- Case context writes — CaseHub-native but couples evolution and review cases
- Signal-driven — clean decoupling but signals decay; configuration should persist
- MCP/REST mutations — circular (devtown calling its own API)
**Rationale:** Most direct, most testable. Workers have access to DevTown's CDI graph.
Each category maps to a specific service call. Matches the "apply" stage in DevTown's
5-stage lifecycle (analyze → propose → review → apply → observe).
**Trade-offs:** Requires those internal services to exist (or be built) in devtown
**Sources:** DevtownProposalSource.java, DevtownCategoryProvider.java
**Exploration:** quick
**Status:** captured
