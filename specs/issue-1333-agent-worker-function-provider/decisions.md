# Design Decisions — #1333 Agent Worker Function Provider

## D1: AgentWorkerFunctionProvider follows the established SPI pattern

**Choice:** New `AgentWorkerFunctionProvider` in `engine-support-core` implements `WorkerFunctionProvider`, handles `agent:` key on YAML worker nodes, and replaces the current inline `agent:` fallback in `YamlCaseDefinitionConverter`.
**Alternatives:**
- Keep inline `agent:` handling in the converter — prevents SPI-based discovery and breaks the provider contract all other worker types follow
- New module for agent integration — over-engineered for a single provider class
**Rationale:** Every other worker type (flow, MCP, A2A, ReAct, pattern, playbook) follows this exact SPI. The agent type should not be special-cased in the converter. Moving it to a provider enables `discoverWorkers()` support and makes the agent worker a first-class citizen of the registry.
**Trade-offs:** The inline `agent:` YAML shape changes — existing YAML that uses `systemPrompt`/`model` directly will need to migrate to eidos descriptors. This is intentional.
**Sources:** `A2AWorkerFunctionProvider.java`, `McpWorkerFunctionProvider.java`, `FlowWorkerFunctionProvider.java`, `WorkerFunctionProvider.java` SPI
**Exploration:** quick
**Status:** captured

## D2: Execution-time agent resolution via AgentRegistry

**Choice:** The provider stores only the slot reference (and optional overrides). At execution time, `AgentWorkerFunctionHandler` resolves the descriptor via `AgentRegistry`, renders the system prompt via `SystemPromptRenderer`, and invokes via `AgentProvider`.
**Alternatives:**
- Parse-time resolution with caching — fast but stale if agent descriptor changes after case load
- Parse-time validation + execution-time resolution — two lookups for marginal benefit
**Rationale:** Consistent with how MCP and A2A connect at runtime (endpoint is a reference, actual connection happens at execution). Agents can be registered, updated, or replaced after case definitions are loaded. Deferred resolution also avoids circular dependency between case loading and agent registration.
**Trade-offs:** No fail-fast validation at YAML load time — a bad slot reference surfaces only at execution. Acceptable because the same is true for MCP endpoints and A2A agent cards.
**Sources:** `AgentRegistry.java` (eidos-api), `AgentWorkerFunctionHandler.java`, `WorkerAgentInvoker.java`
**Exploration:** quick
**Status:** captured

## D3: AgentTaskTarget as a new sealed BindingTarget permit

**Choice:** New `AgentTaskTarget` class implementing `BindingTarget` (sealed interface). Uses capability-based matching via `AgentRegistry.find(AgentQuery)` to resolve a pool of candidate agents, with best-match as the default selection strategy. Modelled after `HumanTaskTarget` but uses eidos capability matching instead of candidate groups.
**Alternatives:**
- Reuse `CapabilityTarget` with agent-aware routing — conflates worker capability routing with agent capability matching
- Extend `HumanTaskTarget` — deprecated, wrong abstraction (human vs. agent)
**Rationale:** Clean separation between human task routing (candidate groups, claim deadlines) and agent task routing (capability matching, availability probing). The sealed interface's exhaustive switch gives compile-time guarantee that all dispatch sites handle the new target type.
**Trade-offs:** Adds a new permit to the sealed `BindingTarget` interface — every exhaustive switch in the codebase must be updated. This is a feature, not a bug: the compiler finds every dispatch site that needs agent-task handling.
**Sources:** `BindingTarget.java`, `HumanTaskTarget.java`, `BindingExecutorResolver.java`, `AgentRegistry.java`
**Exploration:** quick
**Status:** captured

## D4: Scope and default strategy for agent-task: binding

**Choice:** The binding resolver queries `AgentRegistry.find(AgentQuery)` and returns all matching agents (multi-candidate pool). The default selection strategy picks the single best match. Future iterations can add other strategies (round-robin, load-based, etc.).
**Alternatives:**
- Single best-match only — simpler but forecloses future flexibility
- Full strategy framework now — over-engineering for the first iteration
**Rationale:** Build the pool infrastructure now, ship with best-match default. The `AgentRegistry.find()` already returns `List<AgentMatch>` ordered by match quality, so multi-candidate is natural. The selection strategy is a plug point for later.
**Trade-offs:** Initial implementation only exercises the single-best-match path. Pool infrastructure is built but untested with other strategies until they're needed.
**Sources:** `AgentRegistry.find()`, `AgentMatch`, `AgentQuery`
**Exploration:** quick
**Status:** captured
