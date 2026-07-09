# Design Journal — issue-203-context-bridge-protocol

## §Session 1 — 2026-07-09: Design, spec, adversarial review

### Decisions made

- **ContextBridge<T> SPI in engine-api** — not platform-api. Adversarial debate (4 advocates + judge) confirmed every module that calls initialise() already depends on engine-api. Placing it in platform-api would solve a dependency problem that doesn't exist.
- **Reified Varargs Type Token for DSL** — `Worker.builder().<T>fn().apply(fn)` captures runtime type via array component type reification. Zero-argument, compile-time safe, runtime class available for bridge resolution.
- **fn().apply() naming** — `fn` for the type-capturing step, `apply` for the lambda binding. Consistent with Drools RuleBuilder and existing `WorkerFunction.Sync.fn()`.
- **PropagationContext removed from initialise()** — engine threads it independently. Snapshot bridges don't need it; live-view bridges access it via CaseInstance.
- **defaultWorkerBridge (not defaultContextBridge)** — the bridge is about worker input typing, not case context.
- **Five boundary points** — engine workers (fully specified), WorkItems (#689), SubCase (#690), Signals (#691), Connectors (#692). All use the same ContextBridge protocol.
- **initialise() takes JsonNode narrowedInput** — engine evaluates JQ first, bridge receives already-narrowed data. Avoids module-tier violation (JQEvaluator is in engine-common, bridges in engine-api).

### Design review outcome

Two rounds of adversarial review:
- **Round 1** (v1 spec): 16 issues, implementor timed out. Identified circular dependency as critical.
- **Round 2** (v2 spec after fixes): 5 rounds, 18 issues, all 18 verified. $23.20. Spec approved.

### What's next

Implementation plan committed at `docs/plans/2026-07-09-context-bridge-implementation.md`. 8 tasks. Task 1 (WorkerFunction<T> parameterisation) started but reset — must use IntelliJ MCP for all code operations.
