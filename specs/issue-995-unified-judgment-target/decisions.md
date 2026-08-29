## D1: Unified JudgmentTarget with RoutingConfig — delete HumanTaskTarget

**Choice:** JudgmentTarget becomes THE unified yield target. HumanTaskTarget is deleted. Caller-type-specific routing hints move to a `RoutingConfig` sealed interface (nullable) on JudgmentTarget. `HumanRoutingConfig` carries templateRef, candidateGroups, candidateUsers, claimDeadlineHours, payloadType. Yield-semantic fields (title, outcomes, scope, priority) move from HumanTaskTarget to JudgmentTarget directly. `DelegatingJudgmentScheduler` replaces `NoOpJudgmentScheduler` and delegates to `HumanTaskScheduler` when HumanRoutingConfig is present.
**Alternatives:**
- Shared `YieldTarget` interface keeping both types — doesn't unify dispatch/completion/verification paths; two parallel paths persist
- Flatten all fields onto JudgmentTarget as nullable — 20+ fields with no type-level guidance; no extensibility for future caller types
- Keep separate types (status quo from #996 D1) — pragmatic rationale that doesn't hold under "fix the design" directive
**Rationale:** First-principles separation: WHAT (yield semantics on JudgmentTarget) vs WHO (routing hints on RoutingConfig) vs HOW VERIFIED (verifier on JudgmentTarget). Sealed RoutingConfig extensible for future caller types without touching target. Verification, evidence, and provenance apply uniformly to ALL yields. One dispatch path, one completion path, one escalation path.
**Trade-offs:** Significant refactoring across ~6 consumer repos (~25 Java files, ~30 YAML files). Execution via multi-repo work slot with IntelliJ workspace covering all repos — semantic refactoring, never find-and-replace. HumanTaskScheduler SPI preserved behind DelegatingJudgmentScheduler — work repo handler unchanged.
**Sources:** `HumanTaskTarget.java` (16+ fields mixing concerns), `CaseContextChangedEventHandler.java:685-815` (publishHumanTaskSchedule), `BindingTarget.java:27` (sealed permits), Epic #994 (caller-agnostic vision), cross-repo impact analysis (work, clinical, devtown, life, soc, examples, fsitrading)
**Exploration:** deep-analysis
**Status:** captured

## D2: Cross-repo execution via multi-repo work slot

**Choice:** Engine-side design and plan on this branch. Cross-repo refactoring (HumanTaskTarget deletion, consumer migration) executed in a multi-repo work slot with ALL repos in one IntelliJ workspace. Semantic refactoring via ide_refactor_rename, ide_find_references — never grep/find-and-replace.
**Alternatives:**
- Engine-only changes, consumer repos update independently — risks stale consumers, no atomic refactoring
- Find-and-replace across repos — misses overloads, cross-file references, type relationships
**Rationale:** IntelliJ semantic refactoring across a multi-repo workspace is the only safe way to rename/delete types referenced across 6+ repos. One workspace, one refactoring operation, all references updated atomically.
**Trade-offs:** Requires a work slot with 8 repos (engine, work, clinical, devtown, life, soc, examples, fsitrading). Larger workspace but necessary for correctness.
**Depends on:** D1 (what's being refactored)
**Sources:** Cross-repo blast radius analysis, IntelliJ MCP workspace capabilities
**Exploration:** quick
**Status:** captured

## D3: JudgmentEscalator SPI — verification failure handling

**Choice:** `JudgmentEscalator extends NamedStrategy` in `api/spi/judgment/`. `escalate(EscalationContext) → EscalationDecision` sealed: `ReYield(feedback)`, `RouteHigher(minimumTrustLevel)`, `Fault(reason)`. `EscalationContext` carries original yield, failed response, verification result, escalation count. `FaultEscalator` (`@DefaultBean`) always returns Fault. `JudgmentTarget` gains nullable `escalatorStrategy` field. `JudgmentEscalationHandler` resolves escalator and executes the decision. ReYield re-publishes with feedback in prompt. RouteHigher re-publishes with elevated trust threshold. Fault marks PlanItem FAULTED.
**Alternatives:**
- Escalation logic inline in JudgmentCompletedHandler — no pluggability, blocks/qhorus can't provide custom escalation strategies
- Separate escalation handler per strategy — violates NamedStrategy convention, no single dispatch point
**Rationale:** Same NamedStrategy pattern as JudgmentVerifier and ErrorClassifier. Pluggable escalation is the governed yield vision — qhorus E8 (context-aware redistribution) provides a custom escalator, engine provides the SPI and default.
**Trade-offs:** Escalation count tracking needed to prevent infinite re-yield loops. Max escalation depth (default 3) configurable per case.
**Depends on:** D1 (unified target — escalation applies to all yields)
**Sources:** `JudgmentEscalationHandler.java` (current log-only handler), `JudgmentVerifier` (NamedStrategy precedent), Epic #994 (escalation vision), #999 body
**Exploration:** quick
**Status:** captured

## D4: DagNode judgment integration — CompletableFuture blocking

**Choice:** Judgment nodes in DagDriver use a `JudgmentWorkerFunction` that publishes the judgment request and blocks the virtual thread on a `CompletableFuture<JudgmentResponse>`. `JudgmentCompletedHandler` resolves the future. Same pattern as `WorkerRuntime.awaitCase()`. No DagDriver changes — the synchronous `Function<T, R>` contract is preserved. SWF integration via `call: casehub:judgment` callable task type in the flow module.
**Alternatives:**
- DagDriver gains native yield/resume mechanism — invasive change to a general-purpose executor for one use case
- Judgment nodes modeled as choreography bindings within a DAG compound — loses DAG ordering guarantees
**Rationale:** DagDriver's `Function<T, R>` executor runs on virtual threads. Blocking on a CompletableFuture is cheap and correct on virtual threads. The executor doesn't know or care that it's waiting for a judgment — it just blocks until the function returns. No changes to DagDriver, DagPlan, or DagNode.
**Trade-offs:** Timeout enforcement moves to the executor (Future.get with timeout) rather than the DagDriver. If the judgment expires, the executor receives a TimeoutException which the DagDriver handles as a node failure (existing contingency/skip mechanism applies).
**Depends on:** D1 (unified target)
**Sources:** `DagDriver.java` (execute method, Function<T,R>), `WorkerRuntime.awaitCase()` (CompletableFuture pattern), #1000 body
**Exploration:** quick
**Status:** captured

## D5: React module integration tests — scope

**Choice:** Two `@QuarkusTest` classes: `ReActExecutionIntegrationTest` (end-to-end case flow with mock ChatModelProvider) and `ReActAuditTrailTest` (REACT_CYCLE EventLog entries + protocol metadata). Independent of the judgment generalization — uses existing test infrastructure.
**Alternatives:** None — the issue body specifies exactly these two tests.
**Rationale:** Test infrastructure already exists in react module. Pattern follows casehub-engine-a2a integration tests.
**Trade-offs:** None. Pure test addition.
**Sources:** #957 body (test specifications), `react/src/test/resources/application.properties` (existing test config)
**Exploration:** quick
**Status:** captured
