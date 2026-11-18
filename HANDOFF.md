# HANDOFF — casehub-engine

## Last Session

Closed #726 — engine Sync handler delegates to casehub-worker WorkerExecutor. Cross-repo work spanning engine, worker, and agency repos.

### What Landed

4 commits on engine main (SHA `5bd1280da`):

1. **Interface change:** `WorkerFunctionHandler.execute()` and engine's `WorkerExecutor.execute()` now accept `Worker` + `Capability` instead of decomposed fields (`WorkerFunction`, `int timeoutMs`). All 7 handler implementations updated.
2. **ExecutionPolicy enrichment:** `WorkerExecutionOrchestrator` applies the engine's default timeout to the Worker's ExecutionPolicy before passing to handlers. Handlers always see a concrete timeout.
3. **Handler split + delegation:** `SyncAgentWorkerFunctionHandler` split into `SyncWorkerFunctionHandler` (delegates to `io.casehub.worker.runtime.WorkerExecutor` for timeout, retry, schema validation, OTel) and `AgentWorkerFunctionHandler` (keeps engine-local execution with invoker and protocol metadata).

### Cross-Repo Commits

- **casehub-worker** (main): `042b0cb` — ExchangeProcessor support in `DefaultWorkerExecutor.liftToUni()`
- **casehub-agency** (main): `6d8aeaa5` — PatternWorkerFunctionHandler mechanical signature update

### Design Decisions

- Virtual-thread protocol (PP-20260723-c4c1cf) adapted the original issue's approach: `.await().indefinitely()` on virtual thread instead of `runSubscriptionOn()` wrappers.
- `WorkerExecutionContext` no longer exists (replaced by WorkerScope parameter passing). `PolicyEnforcer` not used (worker executor uses SmallRye Guard directly).
- Function wrapping bridges engine WorkerRuntime into worker executor without API changes — the handler wraps the Worker's function in a closure that captures the engine-created runtime.
- Agent handler keeps its own `future.get()` timeout — AgentWorkerFunction has no `WorkerFunction.Sync` fn() for the worker executor to lift.

### Blog

"The Handler That Stopped Reimplementing" — covers execution-concern delegation, function wrapping, and the handler split. SVG diagrams. Promoted to `docs/blog/`.

### Follow-Up Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 1325 | REINVOKED mode test coverage for SyncWorkerFunctionHandler | XS | Low |
| 1326 | Integration test for default timeout enrichment path | XS | Low |
| 1327 | Drop redundant `outputProjection` param from engine WorkerExecutor | XS | Low |
| 1328 | AgentWorkerFunctionHandler — Guard-based timeout and OTel | S | Med |

### Code Review Findings Addressed

- Stale Javadoc on `WorkerFunctionHandler` updated to reflect delegating vs direct timeout
- `PersistentWorkerFunctionHandler` FQN imports cleaned up
- Input type validation duplication documented as intentional (engine exception is more specific)

## Queue Status

4 items remaining in queue: #1325, #1326, #1327, #1328 (all follow-ups from this session).

## Unpublished

Engine has 4 unpublished commits on main. Workspace has 9 unpublished commits. Neither has been pushed to remote.

## Pre-Existing State

- Engine has stashed pre-existing changes from prior work: `git stash list` shows `stash@{0}` (pre-work-end) and `stash@{1}` (pre-726). These are unrelated API/rest/persistence modifications from a different session.
- Workspace has stashed pre-existing blog modifications: 2 blog entries with uncommitted edits.

## References

- `docs/specs/issue-726-sync-handler-delegates-worker-executor/2026-11-18-sync-handler-delegates-worker-executor-design.md`
- `docs/specs/issue-726-sync-handler-delegates-worker-executor/decisions.md` (D1–D6)
- `plans/2026-11-18-sync-handler-delegates-worker-executor.md`
- `docs/blog/2026-11-18-mdp01-the-handler-that-stopped-reimplementing.md`
- PP-20260723-c4c1cf — virtual-thread handler convention
