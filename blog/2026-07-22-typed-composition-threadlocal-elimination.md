# Typed Composition and the Death of ThreadLocal

Every worker function in CaseHub took `Map<String, Object>` and returned `Map<String, Object>`. The type system couldn't tell you what a worker expected or what it produced — that knowledge lived in documentation, in naming conventions, in the heads of the people who wrote it. The generic `WorkerFunction<T>` had one type parameter for input, none for output, and the output was buried inside `WorkerOutcome.Success` where it didn't belong.

We fixed this by adding a second type parameter: `WorkerFunction<T, R>`. Output moves to `WorkerResult<R>` as a top-level record component. `WorkerOutcome<R>` carries the phantom type but no data — it's purely a semantic discriminator (Success, Declined, Failed, Expired). The split is clean: outcome tells you *what happened*, output tells you *what was produced*.

## Three levels of ceremony

The builder DSL gives you exactly the ceremony you need:

```java
// Map → Map (untyped — backwards compatible)
Worker.builder().name("simple").function(input -> WorkerResult.of(Map.of("done", true)))

// OrderInput → Map (typed input, untyped output)
Worker.builder().name("typed-in").<OrderInput>fn().apply((order, scope) ->
    WorkerResult.of(Map.of("total", order.quantity() * order.price())))

// OrderInput → OrderConfirmation (fully typed)
Worker.builder().name("typed-both").<OrderInput>fn().returning(OrderConfirmation.class)
    .apply((order, scope) -> WorkerResult.of(new OrderConfirmation(order.id(), "confirmed")))
```

The `scope` parameter is the interesting part. It replaces `WorkerExecutionContext.currentRuntime()` — the ThreadLocal that let worker functions reach back into the engine to execute other workers or spawn sub-cases.

## Why ThreadLocal had to go

`WorkerExecutionContext` was a static ThreadLocal holding two values: the `WorkerContext` (channels, experiences, case metadata) and the `WorkerRuntime` (execute, spawn, await). Set before the function call, cleared in a finally block. Classic thread-scoped context pattern.

The problem isn't thread safety — virtual threads make that moot. The problem is that ThreadLocal is invisible coupling. A worker function's signature says `Function<Map, WorkerResult>` but its actual dependencies include a runtime it reaches through a static accessor. You can't test the function in isolation without setting up the ThreadLocal. You can't compose functions without ensuring the context is propagated. You can't reason about what the function needs by reading its signature.

`WorkerScope` makes the dependency explicit. It's the second parameter to the BiFunction: `BiFunction<T, WorkerScope, WorkerResult<R>>`. The scope carries `caseId()`, `taskId()`, and `execute()` — everything a worker needs to call other workers. `WorkerRuntime extends WorkerScope` adds engine-specific methods: `context()` for channels and experiences, `spawnCase()` and `awaitCase()` for sub-case orchestration.

The `scope` parameter means worker functions are genuinely pure — their entire dependency surface is in the signature. The engine passes `this` (the runtime) as scope. Tests pass a stub. Sequences get the scope from their own BiFunction parameter instead of fishing it out of a ThreadLocal. Zero `ThreadLocal` in the engine after this change.

## Diagnostic isolation

The `_outcomes` namespace that tracked reroute state (excluded agents, attempt history) is renamed to `_diagnostics`. Same structure, different name — but the name change matters because it groups all per-task diagnostic state under a single filterable key. When input projection runs before a worker invocation, it can strip `_diagnostics.<X>` for all X that aren't the current task's binding. LLM-driven workers see only their own diagnostic history, not their siblings'. The isolation is namespace-based, not permission-based — simple and auditable.

## The numbers

79 files for type propagation, 15 files for ThreadLocal elimination, 5 files for output conversion and YAML support, 6 files for the diagnostic namespace rename. 2,361 tests pass across all modules. The worker-api foundation types were already in place — this branch propagated them through the engine and deleted the ThreadLocal bridge.
