## D1: Budget gate ownership

**Choice:** Decision loops own the budget gate. Remove pool creation and `setBudgetPool()` calls from `SwarmCoordinatorBuilder.buildTopology()`. The topology's single pool in `configure()` is the only pool. Direct consumers (SwarmRoutingStrategy, CollectiveReasoningAggregation, SwarmProvisioningAdvisorImpl) keep their `setBudgetPool()` API for the YAML path but receive no pool in the builder path.
**Alternatives:**
- Consumers get AgentBudgetView on shared pool — requires call-context mechanism to prevent double-counting, adds ~80 lines of complexity for a path that doesn't currently exist
**Rationale:** The decision loop's BudgetAwareSummariser is the natural budget boundary. The builder always creates topologies with decision loops, so consumer-level gating is redundant in this path. The YAML path (PatternCompiler) still needs direct consumer gating until #385 introduces topology-based YAML compilation.
**Trade-offs:** If direct consumers are ever used outside a decision loop (not currently possible through the builder), they'd lack budget protection. Mitigated by the YAML path retaining setBudgetPool().
**Sources:** SwarmCoordinatorBuilder.java:83-121, CentralisedTopology.java:70-124, SwarmRoutingStrategy.java:78-84, PatternCompiler.java:152-154
**Exploration:** quick
**Status:** captured

## D2: Token recording in the topology path

**Choice:** Add `setTokenRecorder(LongConsumer)` to the three direct consumers. In the topology path, `CentralisedTopology.configure()` wires it to `pool.recordTokens()`. Consumers use `tokenRecorder` for recording and `budgetPool` for gating — never both simultaneously. YAML path uses `setBudgetPool()` (gating + recording). Topology path uses `setTokenRecorder()` (recording only, gating handled by BudgetAwareSummariser).
**Depends on:** D1 (single pool means token recording must come from the same pool)
**Alternatives:**
- Skip token recording — accept call-only enforcement (maxCalls=50 is the primary mechanism). Token enforcement was already broken in Pool B (zero-estimator default). Simpler but wastes the existing token enforcement infrastructure.
- Add tokensConsumed to DecisionOutcome — couples domain types to infrastructure, changes sealed interface hierarchy, breaks exhaustive switches
**Rationale:** Clean separation of concerns: BudgetAwareSummariser handles call gating, consumers handle token recording, both operate on the same pool. No double-counting. The LongConsumer callback keeps consumers decoupled from the pool type.
**Trade-offs:** Three consumers gain a new setter method. Consumers must use either `budgetPool` or `tokenRecorder` for recording (not both) — enforced by convention, not compile-time safety.
**Sources:** BudgetAwareSummariser.java:22-24 (default zero estimator), SwarmRoutingStrategy.java:107-108 (existing token recording), AutonomyBudgetPool.java:36-38 (recordTokens ignores correlationId)
**Exploration:** quick
**Status:** captured
