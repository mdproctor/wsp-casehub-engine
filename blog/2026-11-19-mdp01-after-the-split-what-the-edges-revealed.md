---
layout: post
title: "After the Split: What the Edges Revealed"
date: 2026-11-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [code-review, worker-api, execution-policy, testing, interface-design]
series: worker-execution-architecture
---

The handler split separated sync worker execution from agent execution. The diff was the visible work. What came after was more interesting — the code review that asked what the split left exposed.

![What code review surfaced after the handler split](review-findings-after-split.svg)

## The untested execution mode

The sync handler has three modes: TRANSIENT, PERSISTENT, and REINVOKED. REINVOKED is the most complex — it acquires a binding lock from the scoped worker registry, loads accumulated state, passes it to the runtime, executes, then updates the state back. All under a try/finally that guarantees lock release even when execution fails.

None of this was tested.

Every existing test used the two-argument `ExecutionMetadata` constructor, which leaves `executionMode` and `bindingName` null. The REINVOKED conditional never fired. I added three tests: accumulated state round-trip (load from registry, pass to runtime, update from output), lock released on success, and lock released on failure. The failure case matters most — a worker throwing an exception without the try/finally would hold the binding lock indefinitely. The code was correct. But correct-and-untested is an uncomfortable place for a concurrent data structure.

## The redundant parameter

After the interface change added `Capability` as a first-class argument on `WorkerExecutor.execute()`, the separate `outputProjection` parameter was doing nothing the Capability didn't already carry. The orchestrator was calling `capability.outputProjection()` to pass the value. The executor could read it directly.

Removing the parameter simplified the interface from six arguments to five. It also surfaced a contract detail worth knowing: `Capability` enforces `Objects.requireNonNull(outputProjection)` in its compact constructor. The old timeout test passed `null` explicitly for output projection. The new code path gets `"{}"` from the Capability — which still works because `applyOutputSchema` short-circuits on null *output*, not on the projection string itself. The test passes for the right reason, but the contract is tighter than what the test assumed.

## The deliberate "no"

The split left the agent handler with its own timeout via `future.get()` on a virtual thread instead of SmallRye Guard. The question was whether agents should get the same Guard-based timeout, OTel tracing, and retry that sync workers now receive via delegation.

The answer is no, and it is not a gap — it is a boundary.

Agent invocations already capture richer observability than OTel spans: input tokens, output tokens, thinking tokens, cache hits, total cost, latency, backend key, instance ID. Guard-based timeout would layer a second timeout on top of `future.get()` with no benefit. And retry would be harmful — LLM calls cost money per invocation, and the upstream SDK handles transient failures at the HTTP layer. Retrying at the engine level means double-billing for a failure the provider already recovered from.

## What the edges tell you

A refactoring's quality is not in the diff. It is in what becomes visible after the diff lands — untested paths that were always there, redundant parameters that accumulated before the interface settled, and design questions that only have clean answers once the components are separate enough to reason about individually. The split did not create these findings. It made them legible.
