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

## §Session 2 — 2026-07-10: Implementation Tasks 1–7

### Decisions made during implementation

- **BridgeResolver moved to `common/internal/context/`** — initially placed in `runtime`, but `scheduler-quartz` needs it and can't depend on `runtime`. Shared infrastructure that crosses runtime/scheduler module boundary belongs in `common`.
- **WorkerFunction.Sync constructor breaking change** — `Sync(fn)` → `Sync(Class, fn)`. Pre-release platform: fix the design, don't protect callers. ~60 files updated across 3 repos (worker, engine, workers).
- **JacksonPojoBridge uses `FAIL_ON_UNKNOWN_PROPERTIES=false`** — per spec, forward-compatible deserialisation. Added fields deserialise as null; removed fields silently ignored.
- **Pipeline signatures widened to Object** — `WorkerFunctionHandler.execute()` and `WorkerExecutor.execute()` take `Object inputData` instead of `Map<String, Object>`. In-process path (`WorkerRuntime`) stays Map-based per spec.
- **contextBridgeType in EventLog metadata** — written at scheduling time by `WorkerScheduleEventHandler`, read at execution time by `QuartzWorkerExecutionJob` for bridge recovery.
- **evalJqAsJsonNode added alongside evalJqAsMap** — handler evaluates JQ to JsonNode (not Map) for bridge initialisation. Returns full working layer when no inputSchema specified.
- **YAML contextType creates typed Sync placeholder** — `CaseDefinitionYamlMapper` resolves `Class.forName()` at load time, creates `Sync<T>` with UnsupportedOperationException body (external workers have no in-process function).

### Cross-repo changes

| Repo | Branch | Commits | What |
|------|--------|---------|------|
| `casehub-worker` | `issue-203-context-bridge-protocol` | 1 | `WorkerFunction<T>`, `TypedFunctionBuilder`, `Worker.Builder.fn()` |
| `casehub-engine` | `issue-203-context-bridge-protocol` | 7 | Tasks 1–7: full pipeline wiring |
| `casehub-workers` | `main` | 1 | Sync constructor migration (test files only) |

### What's next

Task 8 (integration tests) — the only remaining task. All production code is wired and compiles clean. Tests needed: Pattern 1 (MapBridge identity), Pattern 2 (typed POJO bridge), EventLog metadata verification, backward compatibility with pre-bridge EventLog entries.
