---
layout: post
title: "The Handler That Stopped Reimplementing"
date: 2026-11-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [spi-design, worker-api, delegation, execution-policy, virtual-threads]
series: worker-execution-architecture
---

Five months ago I wrote about where a timeout belongs. The argument was that the executor — not the scheduler adapter — should own the semantic conversion from `TimeoutException` to `WorkerResult.expired()`. Put the concern where it originates.

That reasoning was sound. But I stopped one step short. The engine's sync handler still reimplemented the timeout — `future.get(timeoutMs, TimeUnit.MILLISECONDS)` on a virtual thread — duplicating what the worker executor's SmallRye Guard already handles. Same story for schema validation, OTel tracing, retry with backoff. The worker executor did all of this. The engine handler did some of it, differently, with no way to drift in sync.

The refactoring we landed today finishes the job.

## What the handler was doing

`SyncAgentWorkerFunctionHandler` handled three distinct execution paths in a single class: synchronous worker functions, AI agent invocations, and exchange processors. It pattern-matched on the function type with a switch expression, built its own timeout via `future.get()`, and managed scoped worker sessions. Meanwhile the worker repo's `DefaultWorkerExecutor` — sitting one dependency away — already provided:

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 680 320" style="max-width:680px;width:100%">
  <style>
    text { font-family: 'Inter', 'Segoe UI', system-ui, sans-serif; fill: #334155; }
    .header { font-size: 13px; font-weight: 600; fill: #1e293b; }
    .item { font-size: 12px; fill: #475569; }
    .label { font-size: 11px; font-weight: 600; fill: #64748b; }
    .box { rx: 8; ry: 8; stroke-width: 1.5; }
    .engine-box { fill: #fef3c7; stroke: #d97706; }
    .worker-box { fill: #dbeafe; stroke: #2563eb; }
    .shared-box { fill: #fce7f3; stroke: #db2777; }
    .arrow { stroke: #94a3b8; stroke-width: 1.5; fill: none; marker-end: url(#arrowhead); }
  </style>
  <defs>
    <marker id="arrowhead" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <polygon points="0 0, 8 3, 0 6" fill="#94a3b8"/>
    </marker>
  </defs>

  <!-- Engine handler box -->
  <rect class="box engine-box" x="20" y="20" width="280" height="280"/>
  <text class="label" x="40" y="46">ENGINE HANDLER (BEFORE)</text>
  <text class="item" x="40" y="74">&#x2714; Input type validation</text>
  <text class="item" x="40" y="96">&#x2714; Timeout via future.get()</text>
  <text class="item" x="40" y="118">&#x2714; Exception → WorkerResult</text>
  <text class="item" x="40" y="140">&#x2718; Schema validation</text>
  <text class="item" x="40" y="162">&#x2718; OTel tracing</text>
  <text class="item" x="40" y="184">&#x2718; Retry with backoff</text>
  <line x1="40" y1="200" x2="280" y2="200" stroke="#d97706" stroke-width="0.75" stroke-dasharray="4"/>
  <text class="item" x="40" y="222">&#x2714; WorkerRuntime creation</text>
  <text class="item" x="40" y="244">&#x2714; Scoped session management</text>
  <text class="item" x="40" y="266">&#x2714; Agent invocation + metadata</text>
  <text class="label" x="40" y="290" style="fill:#92400e;font-size:10px">engine-only concerns ↑</text>

  <!-- Worker executor box -->
  <rect class="box worker-box" x="380" y="20" width="280" height="180"/>
  <text class="label" x="400" y="46">WORKER EXECUTOR</text>
  <text class="item" x="400" y="74">&#x2714; Input type validation</text>
  <text class="item" x="400" y="96">&#x2714; Timeout via Guard</text>
  <text class="item" x="400" y="118">&#x2714; Exception → WorkerResult</text>
  <text class="item" x="400" y="140">&#x2714; Schema validation</text>
  <text class="item" x="400" y="162">&#x2714; OTel tracing</text>
  <text class="item" x="400" y="184">&#x2714; Retry with backoff</text>

  <!-- Overlap highlight -->
  <rect class="box shared-box" x="380" y="220" width="280" height="80"/>
  <text class="label" x="400" y="246" style="fill:#9d174d">DUPLICATED</text>
  <text class="item" x="400" y="268" style="fill:#9d174d">Timeout, type check, exception conversion</text>
  <text class="item" x="400" y="286" style="fill:#9d174d">reimplemented in both</text>
</svg>

Three concerns reimplemented. Three more missing entirely in the engine path. Workers dispatched through the engine never got schema validation, never got OTel spans, never got retry — not because those were unwanted, but because the handler was doing the job itself instead of delegating.

## The split

The combined handler was doing two unrelated things: synchronous function execution (which the worker executor already handles) and agent invocation (which it can't, because `AgentWorkerFunction` has no `WorkerFunction.Sync` fn()). After splitting, the distinction is clean:

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 700 260" style="max-width:700px;width:100%">
  <style>
    text { font-family: 'Inter', 'Segoe UI', system-ui, sans-serif; fill: #334155; }
    .header { font-size: 13px; font-weight: 600; fill: #1e293b; }
    .item { font-size: 12px; fill: #475569; }
    .label { font-size: 11px; font-weight: 600; }
    .box { rx: 8; ry: 8; stroke-width: 1.5; }
    .sync-box { fill: #d1fae5; stroke: #059669; }
    .agent-box { fill: #fef3c7; stroke: #d97706; }
    .worker-box { fill: #dbeafe; stroke: #2563eb; }
    .arrow { stroke: #64748b; stroke-width: 1.5; fill: none; marker-end: url(#ah2); }
    .dashed { stroke-dasharray: 6 3; }
  </style>
  <defs>
    <marker id="ah2" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <polygon points="0 0, 8 3, 0 6" fill="#64748b"/>
    </marker>
  </defs>

  <!-- SyncWorkerFunctionHandler -->
  <rect class="box sync-box" x="20" y="20" width="200" height="100"/>
  <text class="label" x="40" y="44" style="fill:#065f46">SyncWorkerFunctionHandler</text>
  <text class="item" x="40" y="68">WorkerRuntime creation</text>
  <text class="item" x="40" y="86">Scoped sessions</text>
  <text class="item" x="40" y="104">Function wrapping</text>

  <!-- Arrow to worker executor -->
  <path class="arrow" d="M 220 70 L 320 70"/>
  <text class="item" x="230" y="62" style="font-size:10px">.await().indefinitely()</text>

  <!-- Worker executor -->
  <rect class="box worker-box" x="320" y="20" width="200" height="100"/>
  <text class="label" x="340" y="44" style="fill:#1e40af">WorkerExecutor</text>
  <text class="item" x="340" y="68">Timeout + retry (Guard)</text>
  <text class="item" x="340" y="86">Schema validation</text>
  <text class="item" x="340" y="104">OTel tracing</text>

  <!-- AgentWorkerFunctionHandler -->
  <rect class="box agent-box" x="20" y="150" width="200" height="90"/>
  <text class="label" x="40" y="174" style="fill:#92400e">AgentWorkerFunctionHandler</text>
  <text class="item" x="40" y="198">Invoker execution</text>
  <text class="item" x="40" y="216">Protocol metadata</text>
  <text class="item" x="40" y="234">Own timeout (future.get)</text>

  <!-- No arrow - self-contained -->
  <path class="arrow dashed" d="M 220 195 L 320 195"/>
  <text class="item" x="260" y="188" style="font-size:10px;fill:#94a3b8">no delegation</text>
  <rect x="320" y="175" width="140" height="30" rx="6" fill="#f1f5f9" stroke="#cbd5e1" stroke-width="1"/>
  <text class="item" x="340" y="195" style="fill:#94a3b8">engine-local</text>
</svg>

`SyncWorkerFunctionHandler` delegates everything the worker executor does well. `AgentWorkerFunctionHandler` keeps its own execution path — the invoker API has nothing in common with `WorkerFunction.Sync`, and agent invocations carry protocol metadata (token counts, cost, latency) that only exists on that path.

## The function-wrapping trick

The worker executor calls `sync.fn().apply(input, null)` — it passes null as the scope because standalone workers don't need one. Engine workers do. They use `WorkerRuntime` for case context, channels, signals, and accumulated state.

The solution: wrap the function before delegation. The handler creates the `WorkerRuntime`, then builds a new `Worker` whose function captures it in a closure:

```java
var originalFn = (BiFunction) sync.fn();
var wrapped = new WorkerFunction.Sync<>(sync.inputType(), sync.outputType(),
    (input, _ignored) -> originalFn.apply(input, runtime));
return new Worker(worker.name(), worker.capabilities(), wrapped,
    worker.executionPolicy(), worker.description(), worker.definitionRef());
```

The worker executor calls the wrapped function. The wrapped function ignores the null scope and uses the engine runtime instead. No API change to the worker repo. No ThreadLocal. The scope provision stays where it belongs — in the engine.

## What changed in practice

The interface change was the prerequisite. `WorkerFunctionHandler.execute()` now takes `Worker` and `Capability` instead of decomposed fields. This propagated through all seven handler implementations (plus one in a sibling repo), but the mechanical changes were trivial — extract `worker.function()` and `worker.executionPolicy().timeoutMs()` where the old parameters were used.

The engine's `WorkerExecutionOrchestrator` enriches the `ExecutionPolicy` with a platform default timeout before passing the Worker down. If a worker definition doesn't specify a timeout, the engine applies one. The handler never needs to resolve defaults — it sees an effective Worker.

What matters is not the lines changed but the concerns eliminated. Every sync worker execution now gets schema validation, OTel spans, and retry support — without the engine reimplementing any of them.
