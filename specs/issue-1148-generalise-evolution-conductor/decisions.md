# Decisions — #1148 Generalise Evolution Conductor

## D1: CapabilityArea IS the HealthSensor abstraction

**Choice:** CapabilityArea already covers health sensing — no new HealthSensor SPI
**Alternatives:**
- Separate HealthSensor SPI — adds a parallel concept when CapabilityArea.assess() already returns healthScore; HealthScoreTracker already aggregates via weighted average
- Merge/rename CapabilityArea — unnecessary churn; the current name and contract work for both readiness methodology and health sensing
**Rationale:** CapabilityArea is a multi-instance SPI with 10 implementations, a registry, and a bootstrap pattern. Trading/AML/Clinical domains register their own CapabilityArea implementations (e.g., TradingRiskCapabilityArea, SharpeRatioCapabilityArea). The existing assess() → healthScore() → weighted aggregation pipeline is domain-agnostic.
**Trade-offs:** Domains must express their health metrics as CapabilityAreaAssessments with a 0.0–1.0 healthScore. Metrics with different scales need normalisation.
**Sources:** CapabilityArea.java:21, HealthScoreTracker.java:47-61, CapabilityAreaRegistry.java, CapabilityAreaBootstrap.java
**Exploration:** quick
**Status:** captured

## D2: ImprovementCategoryProvider SPI for pluggable categories

**Choice:** Multi-instance SPI where each domain registers a provider contributing categories with metadata
**Alternatives:**
- Categories as data on ImprovementConfig — categories stay as strings, each domain sets enabledCategories in YAML; no type safety or metadata
- Category enum SPI — sealed interface per domain; strong typing but couples domains
**Rationale:** Categories are currently hardcoded strings ("dependency-update", "lint-fix", etc.) in ImprovementConfig.effectiveEnabledCategories(). A provider SPI lets domains contribute categories with metadata (id, name, description, domainId) while the engine discovers them at bootstrap via the established CapabilityAreaBootstrap pattern. Default gate mode is handled per-case via the case template's ImprovementConfig.gatePolicy, not per-category — different cases in the same domain can have different gate defaults. Compatible capability areas are implicit from the domain — the domain's categories and capability areas share a domainId.
**Trade-offs:** More infrastructure than pure string config. ImprovementConfig.enabledCategories becomes a filter over provider-contributed categories rather than the source of truth. Category IDs must be globally unique across domains.
**Sources:** ImprovementConfig.java:94-98, CapabilityAreaBootstrap.java, PP-20260921-b7c277 (no @DefaultBean on multi-instance SPIs)
**Exploration:** quick
**Status:** captured

## D3: ImprovementProposalSource SPI for pluggable proposal generation

**Choice:** Multi-instance SPI where domains register sources that generate ImprovementRequests
**Alternatives:**
- Keep signal-based, add domain signals — forces all domains into the consensus model; trading has authoritative metrics, not noisy signals needing consensus
- Replace GoalFormationStrategy entirely — duplicates the shared filtering/budget/conflict logic per domain
**Rationale:** ImprovementGoalFormationStrategy becomes a coordinator that collects proposals from all registered ImprovementProposalSource instances, applies the existing filtering pipeline (budget, conflict, suppression, anti-oscillation), and generates goals. Code-evolution signal-consensus becomes one source.
**Trade-offs:** The signal-consensus model becomes an implementation detail of one source rather than a platform concept. Domains that want consensus can implement it internally.
**Sources:** ImprovementGoalFormationStrategy.java:83-156, SignalRegistry (consensus model)
**Exploration:** quick
**Status:** captured

## D4: RegressionEvaluator SPI delegated from RegressionDetector

**Choice:** RegressionDetector stays as orchestrator; new RegressionEvaluator SPI lets domains define what "worse" means
**Alternatives:**
- Full RegressionDetector SPI — each domain reimplements the monitoring lifecycle (observation windows, revert triggering)
- Keep concrete, extend CapabilityArea — CapabilityArea.assess() already returns healthScore, but regression detection needs comparison logic (baseline vs current) beyond what assess() provides
**Rationale:** RegressionDetector manages the monitoring lifecycle (post-merge observation window, active monitors, revert events). The domain-specific part is "did this change make things worse?" — evaluate(baseline, current) → RegressionVerdict. The detector calls all registered evaluators and aggregates verdicts.
**Trade-offs:** Regression evaluation is decoupled from the monitoring lifecycle. Domains that need different monitoring windows would need to configure RollbackPolicy rather than having their own detector.
**Sources:** RegressionDetector.java:87-112 (checkActiveMonitors), RegressionDetector.java:114-139 (onMetricsDegraded)
**Exploration:** quick
**Status:** captured

## D5: Code-evolution implementations stay in runtime-core

**Choice:** No new Maven module — code-evolution implementations remain in runtime-core as defaults
**Alternatives:**
- Extract to engine-evolution-code module — clean separation but adds module complexity; a bare engine has no evolution capability
- Extract to blocks module — crosses engine/blocks boundary prematurely
**Rationale:** Pre-release platform — the cost of a module split is not justified yet. Code-evolution implementations are the default set. Trading/AML domains add their implementations alongside (multi-instance SPIs coexist). If module separation becomes warranted later, the SPI boundaries make extraction mechanical.
**Trade-offs:** runtime-core contains both domain-agnostic orchestration and code-evolution-specific implementations. They're distinguished by which SPI they implement, not by module boundary.
**Sources:** runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/ (10 capability areas)
**Exploration:** quick
**Status:** captured

## D6: Worker infrastructure IS the executor — no ImprovementExecutor SPI

**Choice:** Existing case lifecycle + WorkerProvider SPI covers execution
**Alternatives:**
- ImprovementExecutor SPI — bypasses case/worker lifecycle; less platform integration
- Hybrid executor-as-worker-adapter — adds an adapter layer for no clear gain
**Rationale:** The case lifecycle already orchestrates improvement execution. A trading domain defines ParameterAdjustmentWorker, ThresholdRecalibrationWorker, etc. through the existing WorkerProvider SPI. The improvement case template determines which workers run. No new execution SPI needed.
**Trade-offs:** Domains must express execution as workers within the case lifecycle. Lightweight parameter changes still get wrapped in a case. This is a feature (auditability) not a bug.
**Sources:** ImprovementGoalFormationStrategy.java:105 (goalFormationService.propose), WorkerProvider SPI
**Exploration:** quick
**Status:** captured

## D7: Direct proposals bypass signal-consensus model

**Choice:** ImprovementProposalSource.propose() returns ImprovementRequests directly; signal-consensus is not required
**Alternatives:**
- Wrap in signals — forces all domains into multi-source consensus model; trading has authoritative metrics
- Configurable per source — maximum flexibility but adds complexity to the filtering pipeline
**Rationale:** The filtering pipeline (budget, conflict, suppression, anti-oscillation) still applies to all proposals regardless of source. Signal-consensus becomes an implementation detail of the code-evolution source, not a platform requirement. Trading domains have authoritative metrics — a single health sensor detecting Sharpe degradation is sufficient to propose.
**Trade-offs:** No built-in corroboration requirement for direct proposals. Domains that want corroboration implement it in their source. The anti-oscillation filter (RollbackHistory check) provides some safety net.
**Sources:** ImprovementGoalFormationStrategy.java:87 (consensusSignals), ImprovementGoalFormationStrategy.java:110-116 (rollback check)
**Exploration:** quick
**Depends on:** D3 (ImprovementProposalSource SPI)
**Status:** captured

## D8: Domain-contributed improvement stages

**Choice:** ImprovementStage becomes a string identifier; domains define stage sequences via ImprovementCategoryProvider
**Alternatives:**
- Keep enum, extend — add generic stages (VALIDATE, DEPLOY, OBSERVE); less flexible, breaks existing enum consumers
- Two-level generic+domain — generic lifecycle phases as enum with domain sub-stages as strings; adds conceptual complexity
**Rationale:** Current ImprovementStage enum has code-evolution-specific stages (SUBMIT_PR, PR_REVIEW). Trading stages (BACKTEST, CROSS_REGIME_VALIDATION, REGULATORY_CHECK) don't map to these. Making stages strings with domain-defined sequences means GatePolicy maps against domain-specific stage identifiers. The gate checkpoint concept stays — domains declare which of their stages are gate checkpoints.
**Trade-offs:** Lose compile-time exhaustiveness on stage matching. Stage validation moves from the compiler to runtime checks. Gate checkpoint determination moves from ImprovementStage.isGateCheckpoint() to the category provider's stage definition.
**Sources:** ImprovementStage enum (INTROSPECT through OUTCOME_RECORDING), GatePolicy.java (uses ImprovementStage in Map keys)
**Exploration:** quick
**Depends on:** D2 (ImprovementCategoryProvider SPI)
**Status:** captured
