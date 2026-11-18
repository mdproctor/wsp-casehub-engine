## D1: Interface scope — full WorkerFunctionHandler interface change

**Choice:** Change `WorkerFunctionHandler.execute()` and engine's `WorkerExecutor.execute()` to accept `Worker` + `Capability` instead of decomposed fields (`WorkerFunction`, `int timeoutMs`).
**Alternatives:**
- Sync-only internal delegation — smaller blast radius but leaves the decomposed interface as tech debt
- Interface change without delegation — changes the contract but misses the execution-concern unification
**Rationale:** The Worker aggregate carries the execution contract (function, policy, identity). Decomposing it strips information handlers need. All 7 handlers + PatternWorkerFunctionHandler (agency repo) update signatures; only Sync handler gets substantive delegation logic.
**Trade-offs:** Cross-repo impact (casehub-agency PatternWorkerFunctionHandler). All handler tests update.
**Sources:** `WorkerFunctionHandler.java:34-44`, `WorkerExecutionOrchestrator.java:310-316`, issue #726 §1-2
**Exploration:** quick
**Status:** captured

## D2: Delegation model — full delegation to worker executor

**Choice:** `SyncWorkerFunctionHandler` injects `io.casehub.worker.runtime.WorkerExecutor` and delegates to it. Worker executor handles timeout (Guard), retry, schema validation, OTel tracing. Engine handler wraps with WorkerRuntime creation and scoped session management.
**Alternatives:**
- Partial delegation — only delegate function invocation + schema validation, keep timeout in engine handler
- No delegation / just split — avoid cross-repo dependency, rework timeout but don't add casehub-worker-runtime
**Rationale:** Eliminates duplicated execution concerns. Worker executor already handles timeout, retry, schema validation, OTel — reimplementing these in the engine is maintenance burden and divergence risk.
**Trade-offs:** New Maven dependency (casehub-worker-runtime) in engine runtime module. Worker executor returns `Uni<WorkerResult>` — on virtual thread, call `.await().indefinitely()`.
**Sources:** `DefaultWorkerExecutor.java` (worker repo — Guard, SchemaValidator, OTel), virtual-thread protocol PP-20260723-c4c1cf
**Exploration:** quick
**Depends on:** D1 (Worker + Capability available on the interface)

## D3: Handler split — Sync+ExchangeProcessor handler + Agent handler

**Choice:** Split `SyncAgentWorkerFunctionHandler` into `SyncWorkerFunctionHandler` (handles `WorkerFunction.Sync` + `ExchangeProcessor`) and `AgentWorkerFunctionHandler` (handles `AgentWorkerFunction`).
**Alternatives:**
- Three-way split — separate ExchangeProcessor handler; maximum separation but Sync and ExchangeProcessor share almost everything
- No split — delegate from within the combined handler based on function type; Agent path skips delegation
**Rationale:** After delegation, Sync (delegates to worker executor with Guard, schema validation, OTel) and Agent (engine-local invocation, protocol metadata, token tracking) share almost nothing. The handler chain's `supports()`/`execute()` pattern exists for polymorphic dispatch — internal switch statements defeat it.
**Trade-offs:** Two classes where one existed. Test coverage must span both.
**Sources:** `SyncAgentWorkerFunctionHandler.java:59-63` (supports), `SyncAgentWorkerFunctionHandler.java:118-145` (function dispatch), issue #726 §3
**Exploration:** quick
**Depends on:** D1 (interface change enables clean split)

## D4: Timeout reconciliation — enrich ExecutionPolicy at call site

**Choice:** In `WorkerExecutionOrchestrator`, apply the engine's default timeout to the Worker's `ExecutionPolicy` before passing to the executor, when `executionPolicy().timeoutMs()` is null.
**Alternatives:**
- Keep timeoutMs on the interface alongside Worker/Capability — redundant but safe
- Move default into worker executor — requires changing worker-runtime API
**Rationale:** Clean separation: engine decides the effective policy, worker executor enforces it. The enrichment happens once at the orchestrator, not inside each handler. No change to worker-runtime API.
**Trade-offs:** Worker passed to handler may differ from the Worker in the registry (enriched policy). Must document that handlers see the effective Worker, not the definition-time Worker.
**Sources:** `WorkerExecutionConfig.java:26-28`, `WorkerExecutionOrchestrator.java:272`
**Exploration:** quick
**Depends on:** D2 (delegation model requires the worker executor to receive a concrete timeout)

## D5: Delegation architecture — handler-internal delegation

**Choice:** Engine's `DefaultWorkerExecutor` (composite) still iterates handlers. The new `SyncWorkerFunctionHandler` internally injects `io.casehub.worker.runtime.WorkerExecutor` and delegates to it. Handler chain architecture stays intact.
**Alternatives:**
- Composite bypass — engine's DefaultWorkerExecutor detects Sync and delegates directly, bypassing handler chain; couples composite to function-type knowledge
- Dual executor — engine's WorkerExecutor wraps worker executor for all types, handlers become decorators; bigger refactoring
**Rationale:** Preserves the handler-chain architecture with minimal blast radius. Only one handler gets the new dependency. Other handlers are unaware of the delegation.
**Trade-offs:** One extra level of dispatch for Sync functions (composite → handler → worker executor). Negligible at runtime.
**Sources:** `DefaultWorkerExecutor.java:72-80` (handler iteration), issue #726 §4
**Exploration:** quick
**Depends on:** D2 (full delegation model), D3 (Sync handler is the delegation point)
