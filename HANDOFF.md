# HANDOFF — casehub-engine

## Last Session

Designed and implemented #1333 (agent worker function provider for eidos dispatch). Full brainstorm → spec → plan → implementation → review cycle. Filed three follow-up issues (#1338, #1339, #1340) for the remaining agent-task wiring.

### What Was Built

Branch `issue-1333-agent-worker-function-provider` — 8 commits (6 feature + 2 fixes found in review):

1. **`AgentSlotConfig`** — new record in `api/model/ai/` holding slot reference + optional overrides (model, userMessageTemplate, inputProjection, outputProjection).
2. **`AgentWorkerFunction` dual-mode** — the existing record gains a second construction path: `eager(invoker)` (backward-compatible) and `ofSlot(config)` (deferred resolution). `isDeferred()` distinguishes the modes.
3. **`AgentTaskTarget`** — new sealed `BindingTarget` permit for capability-based agent dispatch. All exhaustive switch sites updated (CaseDefinitionService, CompensationGraphProjection, BindingExecutorResolver). `Binding.Builder.agentTask()` convenience method added.
4. **`AgentWorkerFunctionProvider`** — in `engine-support-core`, handles `agent:` YAML with `slot:` child. Creates `AgentWorkerFunction.ofSlot(config)`.
5. **`AgentWorkerFunctionHandler` extended** — resolves slot at execution time: `AgentRegistry.find(AgentQuery.bySlot(...))` → `AgentDescriptor` → `SystemPromptRenderer.render()` → `WorkerAgentInvoker`. Existing eager path unchanged.
6. **Migration** — inline `agent:` fallback removed from `YamlCaseDefinitionConverter`. `YamlAgent` codegen record gained `slot`, `userMessage`, `inputProjection`, `outputProjection` fields.

### Review Findings Fixed

- `findById(slot, tenancyId)` was wrong — `findById` takes `agentId`, not `slot`. Fixed to `AgentRegistry.find(AgentQuery.bySlot(...))`.
- `BindingExecutorResolver` had a `default` branch silently absorbing `AgentTaskTarget`. Added explicit case.

### Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 1338 | YAML parser for agent-task: binding target | S | Low |
| 1339 | agent-task: runtime dispatch handler for AgentRegistry capability matching | M | Med |
| 1340 | Integration test: slot-based agent worker with CDI wiring | S | Low |

### Blog

"The Agent That Found Its Own Slot" — at `docs/blog/2026-11-22-mdp01-the-agent-that-found-its-own-slot.md`.

## Active Work

#1333 is marked done on branch `issue-1333-agent-worker-function-provider`. Not yet landed on main — needs `work end` to land, squash, and close.

Next in queue: #1336 (integration test: auto-discovered YAML wires do: block workers), then #1338, #1339, #1340.

## Pre-Existing State

Engine has 5 uncommitted formatting changes from a prior session in `rest-core/` and `rest/` (EventLogService, PlanService, PlanViewMapper, EvolutionMcpAdapter, EvolutionMcpAdapterTest). These predate #1333 work and cause spotless failures during `mvn install`. They are NOT part of the current branch work. Stash or commit them before landing.

Engine also has stashed changes from earlier sessions (multiple stash entries).

## References

- `wksp/specs/issue-1333-agent-worker-function-provider/2026-11-20-agent-worker-function-provider-design.md`
- `wksp/specs/issue-1333-agent-worker-function-provider/decisions.md`
- `wksp/plans/2026-11-20-agent-worker-function-provider.md`
- `docs/blog/2026-11-22-mdp01-the-agent-that-found-its-own-slot.md`
- Slot 226: `/Users/mdproctor/claude/casehub/slots/226` (#1332 CollectionStore — still active)
