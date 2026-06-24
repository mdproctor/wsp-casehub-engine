# When Records Replace Classes

The worker primitives migration — Worker, Capability, WorkerFunction, WorkerResult, WorkerOutcome, and three governance types — moved from mutable classes in engine-api to Java records in two foundation dependencies. The change touched every module in the engine. What made it interesting wasn't the scale but what the record contract enforced that the class contract didn't.

## The Type System as Enforcer

The old `PlannedAction` carried `workerId` and `caseId` fields that the engine enriched before passing to the classifier. Workers created a PlannedAction without identity, the engine called `withIdentity()` to fill it in, and the classifier received the complete object. Two problems: the worker could see identity fields it shouldn't care about, and nothing prevented someone from calling `classify()` with an un-enriched action.

The new design splits this cleanly. `PlannedAction` in worker-api carries only what the worker declares: description, action type, parameters. Identity moves to `ClassificationContext` — a separate record constructed by the engine at the classify call site. The classifier receives both, and neither type can exist in a half-populated state. The compiler enforces what was previously a runtime contract.

Same pattern with `WorkerResult`. The old version had `plannedAction` as a nullable field with a runtime check: if the outcome wasn't `Success`, `plannedAction` had to be null. The record version puts `PlannedAction` on `WorkerOutcome.Success` directly. You can't declare a consequential action on a failed outcome — the type won't let you construct it.

## What Records Won't Let You Get Away With

Three things bit us during the migration that the old mutable classes silently accepted:

**Null schemas.** The old `Capability` accepted `null` for `inputSchema` and `outputSchema`. Dozens of tests used `new Capability("cap", null, null)` as a throwaway. The record's compact constructor calls `Objects.requireNonNull` — every one of those tests failed at runtime, not at compile time. The fix is trivial (`Capability.of("cap", "{}", "{}"`), but the failure mode is disorienting: a `NullPointerException` deep in a record constructor, with no compile-time warning.

**Zero retries.** The old `RetryPolicy` accepted `maxAttempts = 0` as the no-retry idiom. The platform-api version validates `maxAttempts >= 1` — zero is semantically invalid. `ExecutionPolicy.noRetry()` is the replacement. The particularly nasty part: because `@QuarkusTest` classes share a container, one bean constructing `RetryPolicy(0, ...)` crashes every test in the module. The stack trace points at a CDI bean's `getDefinition()` method, not at the specific `RetryPolicy` call.

**Lambda ambiguity.** `Worker.Builder` has `function(WorkerFunction)` and `function(Function<Map, WorkerResult>)`. A lambda matches both. The old builder had typed overloads — `function(Agent)`, `function(Workflow)`, `function(Function)` — that disambiguated naturally. The record-based builder doesn't know about Agent or Workflow (they're engine types, not foundation types), so it has the two generic overloads. The fix: `new WorkerFunction.Sync(input -> ...)`.

## What This Enables

With Worker, Capability, and the execution policy types living in foundation modules, consumer repos can work with workers without depending on the engine. A connector that provisions workers only needs `casehub-worker-api` — it never sees the engine's case management machinery. The same types that define a worker's identity and function are the types that consumers interact with at runtime.

AgentDescriptor moved from Worker to CaseDefinition. Workers are now pure foundation-tier records with no eidos dependency. The association between a worker and its AI agent identity is a build-time concern set by the case definition author, not a field on the worker itself. This matters for the platform — worker-api is foundation, eidos is optional. Coupling them was an early convenience that became a dependency constraint.
