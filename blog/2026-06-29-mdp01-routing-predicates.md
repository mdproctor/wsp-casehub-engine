---
layout: post
title: "Routing predicates and the art of not leaking"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [spi, routing, worker-execution, composite, design-review]
---

The composite `WorkerExecutionManager` (#461) solved CDI ambiguity when co-deploying multiple worker backends. But it left a question hanging: what happens when Quartz — the catch-all fallback — catches a capability it can't actually execute?

External workers (HTTP, MCP, Camel, Script, GitHub Actions) carry placeholder `Sync` functions that return empty maps. If an external backend goes down and Quartz catches the capability, it runs the placeholder. No error. No warning. Wrong output, silently.

The fix looks obvious: add `WorkerFunction.None` so external workers say what they are, then teach Quartz to reject them. But the routing predicate `supports(String capabilityName, String tenancyId)` doesn't carry function-type information. My first instinct was to change the signature — `supports(Worker, Capability, tenancyId)` — giving backends everything they need.

I talked myself into it with clean arguments: the information is already available at the routing layer, backends that don't need `Worker` ignore it, breaking changes cost nothing externally. But when I stepped back, the question was sharper: does a scheduler need domain objects? Quartz is a job scheduler. It fires timers and manages retries. Passing `Worker` and `Capability` into its routing predicate leaks orchestration concepts into the scheduler tier — the exact coupling we're trying to remove by making Quartz replaceable.

The right decomposition was two methods, each with one responsibility. `supports()` stays as capability+tenant routing — what external backends need. `canExecute(WorkerFunction)` is the new additive method — what in-process backends need. The routing strategy checks both. Recovery (which has no `Worker`) uses only `supports()`. Clean separation.

The design review caught something I'd missed. My initial `canExecute()` on Quartz was `return !(function instanceof None)` — a negative check that fails open for unknown `WorkerFunction` types. Since `WorkerFunction` is a non-sealed interface, any module can add a new variant. The negative check would silently accept it. Claude flagged this and proposed positive handler delegation instead: iterate the actual `WorkerFunctionHandler` chain and return `true` only when a handler explicitly supports the function. Unknown types are rejected by default. Fail-safe, not fail-open.

The review also killed my broadcast idea for `schedulePersistedEvent()` — I'd proposed broadcasting recovery events to all backends instead of routing them, arguing it was safe because only Quartz implements recovery. True, but broadcasting means Quartz creates spurious jobs for external workers whose recovery events should be silently consumed by their own backend. The routed approach is correct.

Four issues, ten files, 232 lines. `WorkerFunction.None` in the foundation tier. `canExecute()` as an additive SPI method. Non-blocking startup recovery. Trigger method documentation. The composite WEM is now precise about what each backend can handle — Quartz no longer pretends it can execute everything.

The pattern here generalises: when a predicate needs information from a different tier, don't widen the signature — add a second predicate at the right level of abstraction and compose them at the routing layer.
