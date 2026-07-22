---
title: "Closing the Bridge, Opening the Types"
date: 2026-07-22
entry_type: note
subtype: diary
tags: [contextbridge, typed-composition, worker-api, architecture]
---

# Closing the Bridge, Opening the Types

Two arcs today. The first one finished; the second one started something I've wanted to do for a while.

## The last three boundaries

The ContextBridge protocol was missing three pieces: the connector boundary (#692), linked data references (#740), and action gate resolution typing (#742). None were individually complex, but together they closed the protocol — every execution boundary in the engine now has typed context.

The connector work was the most interesting. I'd expected to create a new `ConnectorTarget` binding type. Tracing it from first principles changed my mind. Bindings are reactive — they fire when context changes, pushing work outward. Connectors are the opposite: external data arriving inward. Different direction, different abstraction. What we built instead is an `InboundSignalBridge` that observes `InboundMessage` events from `casehub-connectors` and routes them to typed case signals. No new binding type. No HTTP endpoints in the engine. The connector repo owns reception; the engine owns typing.

DataRef was the conceptual extension — store a reference instead of the object, resolve on demand. The design review pushed for deferred resolution (resolve at execution time, not scheduling time) which keeps references in EventLog and avoids blocking on the Vert.x event loop. The `$dataRef` discriminator makes references unambiguous in JSON without schema consultation.

Five pre-existing CI failures surfaced along the way — stale contract tests, a missing Flyway migration, API drift between repos. None from our work, all from accumulated cross-session drift. Fixed them all. CI green.

## Map is not a type

The second arc started from a different question. We were designing typed in-process composition (#693) — making `WorkerRuntime.execute()` and `sequence()` work with typed workers instead of just Maps. The obvious design: add a typed execute overload, convert between Maps and POJOs at step boundaries.

Then I looked at WHY Map was everywhere and didn't like what I found. `WorkerResult.output()` returns `Map<String, Object>`. That's the root. Everything downstream — the sequence accumulator, the context application, the output handling — is Map because the result forces it. Workers that naturally produce a `RiskAssessment` must wrap it in `Map.of("risk", assessment.score())` for no architectural reason.

The fix: `WorkerResult<R>`. The output type becomes a generic parameter, declared at the worker definition. Three levels of ceremony — Map→Map workers declare nothing, typed-input workers declare `contextType`, fully typed workers declare both `contextType` and `outputType`. Same three levels in YAML. Map stops being special; it's one possible output type.

This pulled on a thread. `WorkerFunction<T>` became `WorkerFunction<T, R>`. `WorkerOutcome` gained `<R>`. The Async variant — zero engine references, superseded by virtual threads — got deleted.

## ThreadLocal has to go

The other half of #693 was about how workers access their execution context. `WorkerExecutionContext` uses `ThreadLocal` to pass the runtime and context to worker functions as ambient state. The function signature is `Function<T, WorkerResult>` — one parameter, no room for the runtime.

I don't like ThreadLocal. Invisible dependencies, manual stack management in nested calls, testing pain. The fix: `BiFunction<T, WorkerScope, WorkerResult<R>>`. The runtime is the second parameter. Simple workers that don't need it use the single-arg `.apply()` overload — the builder wraps to a BiFunction that ignores the second arg. Workers that compose use the two-arg version and get the runtime explicitly.

`WorkerScope` sits in `casehub-worker-api` (tier 1) with just `caseId()`, `taskId()`, `execute()`. `WorkerRuntime` extends it in engine-api (tier 2) adding `context()`, `spawnCase()`, `awaitCase()`. No circular dependency. The engine always passes a `WorkerRuntime`; workers that need context cast to it.

The foundation types are changed and published. The engine compiles clean against them — Java's raw type compatibility means nothing broke. The behavioral changes (removing the ThreadLocal, wiring the explicit runtime through handlers) are next session's work.

## Where this lands

After this session: ContextBridge protocol complete across all five boundaries. `WorkerFunction<T, R>` and `WorkerResult<R>` published. `WorkerScope` exists. Zero ThreadLocal is one session away.

The context isolation work (#698) — namespacing diagnostic state so sibling tasks don't contaminate each other — is designed and spec'd but not yet implemented. It builds naturally on the `taskId()` that `WorkerScope` now carries.
