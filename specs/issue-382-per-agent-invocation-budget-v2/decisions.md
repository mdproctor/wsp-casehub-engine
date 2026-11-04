## D1: Budget gate ownership

**Choice:** Decision loops own the budget gate. Remove pool creation and `setBudgetPool()` calls from `SwarmCoordinatorBuilder.buildTopology()`. The topology's single pool in `configure()` is the only pool. Direct consumers (SwarmRoutingStrategy, CollectiveReasoningAggregation, SwarmProvisioningAdvisorImpl) keep their `setBudgetPool()` API for the YAML path but receive no pool in the builder path.
**Alternatives:**
- Consumers get AgentBudgetView on shared pool — requires call-context mechanism to prevent double-counting, adds ~80 lines of complexity for a path that doesn't currently exist
**Rationale:** The decision loop's BudgetAwareSummariser is the natural budget boundary. The builder always creates topologies with decision loops, so consumer-level gating is redundant in this path. The YAML path (PatternCompiler) still needs direct consumer gating until #385 introduces topology-based YAML compilation.
**Trade-offs:** If direct consumers are ever used outside a decision loop (not currently possible through the builder), they'd lack budget protection. Mitigated by the YAML path retaining setBudgetPool().
**Sources:** SwarmCoordinatorBuilder.java:83-121, CentralisedTopology.java:70-124, SwarmRoutingStrategy.java:78-84, PatternCompiler.java:152-154
**Exploration:** quick
**Status:** captured
