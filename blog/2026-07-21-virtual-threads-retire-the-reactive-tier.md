# Virtual Threads: Retiring the Reactive Tier

The CaseHub engine has 152 files importing Mutiny's `Uni`. Every `@ConsumeEvent` handler returns `Uni<Void>`, chains operations with `.chain()`, handles errors with `.onFailure().recoverWithUni()`, and fans out parallel work with `Uni.combine().all().unis()`. All of them already run on worker threads — `blocking = true` on every annotation. The reactive machinery adds complexity without protecting IO threads, because we're not on IO threads.

Virtual threads change the equation. A virtual thread blocks without holding a platform thread. That means blocking code — sequential, imperative, try-catch — is correct for the same use cases where we'd been writing reactive chains. The chains were never buying us non-blocking IO; they were buying us compositional structure at the cost of readability and debuggability.

## The pilot

We picked `CaseContextChangedEventHandler` — the heaviest Uni user in the engine at 907 lines. It touches everything: CBR retrieval, routing strategy selection, worker provisioning, humanTask candidate resolution, subcase dispatch. If the conversion pattern works here, it works everywhere.

The mechanical part was straightforward. `@ConsumeEvent(blocking = true)` becomes `@RunOnVirtualThread @ConsumeEvent`. The method returns `void` instead of `Uni<Void>`. Sequential calls replace `.chain()`. Try-catch replaces `.onFailure().recoverWithUni()`. For the fan-out path — dispatching multiple bindings in parallel — we injected the `@VirtualThreads ExecutorService` that Quarkus provides and used `CompletableFuture.allOf()`. `StructuredTaskScope` would be cleaner, but it's still in preview after seven rounds across JDK 21–27. `CompletableFuture` with virtual threads is the pragmatic choice.

## The scope question

The initial plan was to keep the Uni-returning SPIs unchanged and call `.await().indefinitely()` on them from the virtual thread. Claude challenged this: if the root handler is on a virtual thread and returns void, why is Uni still in the call chain? `.await().indefinitely()` scattered through the method body isn't a conversion — it's wrapping.

That shifted the scope from one handler to seven SPI interfaces: `LoopControl`, `AgentRoutingStrategy`, `HumanTaskRoutingStrategy`, `CandidateSetStrategy`, `ImplementationRoutingStrategy`, `PlanningStrategy`, and `StageLifecycleEvaluator`. Every one returned `Uni<T>`. Every implementation — 16 across three repos — was either wrapping a synchronous result in `Uni.createFrom().item()` or using `.emitOn(workerPool)` to move blocking work off the calling thread. On virtual threads, both patterns are unnecessary. The blocking work runs directly; the virtual thread handles the rest.

We changed all seven interfaces to return plain values. Traced every implementation and caller across engine, blocks, and quarkmind. The ripple touched 46 files in the engine alone, plus 8 across the two consumer repos. 2149 tests passing, zero failures.

## What stays reactive

Not everything converts. Three use cases genuinely need non-blocking, event-driven patterns:

**Long-running continuations.** Worker provisioning fires a request and waits for a callback — the provisioner spins up a Claudony agent or a Docker container, and minutes later a completion event arrives. Blocking a virtual thread for minutes waiting for a human or external system is wrong even if it's cheap. The fire-and-callback pattern stays.

**Reactive messaging.** Kafka and AMQP channels use SmallRye's `Multi<T>` API. These are streaming with backpressure — a fundamentally different model from request-scoped handler execution.

**PostgreSQL LISTEN/NOTIFY.** The async broadcasting protocol is non-blocking by nature. Converting it to blocking would mean polling, which defeats the purpose.

Everything else — the remaining ~40 `@ConsumeEvent` handlers, the routing strategies, the orchestration SPIs — follows the pattern this pilot established. The blocking repository implementations already exist alongside the reactive ones. Handlers switch from injecting `Reactive*Repository` to the blocking counterpart. Once no code injects the reactive variants, the reactive interfaces and their implementations get deleted.

## The pattern

For any handler migrating from Mutiny to virtual threads:

1. `@RunOnVirtualThread @ConsumeEvent` — return `void`
2. Sequential blocking code — try-catch for error recovery
3. Fan-out with `@VirtualThreads ExecutorService` + `CompletableFuture.allOf()` when parallel dispatch is needed
4. Inject blocking SPIs directly — no reactive-to-blocking bridging
5. Delete the Uni path — don't wrap it

The last point is the one that matters. Wrapping reactive code in `.await().indefinitely()` from a virtual thread compiles and runs correctly. But it leaves the reactive machinery in place — the interfaces, the implementations, the imports, the mental model. The goal isn't to make blocking calls to reactive APIs. The goal is to remove the reactive APIs from code paths that don't need them.
