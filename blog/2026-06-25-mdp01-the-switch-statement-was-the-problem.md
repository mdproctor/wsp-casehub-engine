---
layout: post
title: "The Switch Statement Was the Problem"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [architecture, spi, serverlessworkflow, worker-execution]
---

## The Switch Statement Was the Problem

CaseHub's `DefaultWorkerExecutor` had a switch statement that routed worker functions to their execution strategy — sync lambdas to a virtual thread pool, agent functions to the AI model, workflow functions to the Serverless Workflow SDK. Three cases, three code paths.

The problem wasn't the switch. The problem was that the switch forced the executor to _know_ every execution technology at compile time. Adding `FlowWorkerFunction` meant importing `Workflow` from the serverlessworkflow SDK. That import dragged the SDK into the runtime module, which depended on common, which depended on api — and suddenly four modules carried a dependency they never used. They were transporting a type, not computing with it.

The `module-tier-structure` protocol names `io.serverlessworkflow.*` as a specific example of an SDK type that should not appear in SPI signatures. Our `WorkflowExecutor` interface in common did exactly that — `execute(Workflow, ...)` right there in the method signature.

### Two of three subtypes were lying

While tracing the dependency leak, I noticed a deeper smell. `WorkerFunction` defined an `execute(Map)` method. `Sync` implemented it honestly. `AgentWorkerFunction` and `FlowWorkerFunction` both threw `UnsupportedOperationException` — two of three subtypes violated Liskov substitution. The `execute()` method was dead in production; `DefaultWorkerExecutor` pattern-matched on the type and called type-specific accessors instead. The interface method existed for nobody.

### Handlers replace the switch

We replaced the hardcoded dispatch with two SPIs. `WorkerFunctionHandler` (in common) declares `supports()` and `execute()`. Each module contributes a handler for its own function types. `SyncAgentWorkerFunctionHandler` in runtime handles sync and agent functions. `FlowWorkerFunctionHandler` in the flow module handles workflows. `DefaultWorkerExecutor` iterates the handlers and applies output schema evaluation as post-processing — it no longer knows what a workflow is.

`WorkerFunctionProvider` (in api) does the same for construction. The YAML mapper delegates SDK-dependent function construction to registered providers. The flow module registers one for `do:` blocks. Agent and sync construction stay inline in the mapper — they carry no external SDK dependency.

`FlowWorkerFunction` moved from api to the flow module. The serverlessworkflow SDK stays exclusively there.

### What got deleted

`WorkflowExecutor` SPI from common. `NoOpWorkflowExecutor` from runtime. `FlowWorkerExecutor` from flow. The `execute()` method from `WorkerFunction` itself — it becomes a marker interface, and the type system no longer pretends functions can self-execute when they can't. `WorkerFunction` in `casehub-worker-api` is now a pure marker: "this is something a worker runs." How it runs is the handler's concern.

The one thing I pushed back on during design was carrying `JsonNode` as the intermediate type between modules. It would have worked — but we don't pass untyped JSON blobs around anywhere else in the platform, and starting now would have been a smell in its own right. The provider model avoids it: raw YAML enters at the factory boundary, the provider produces a typed function, and the `JsonNode` never travels further.
