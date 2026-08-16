## D1: Target type for timer-triggered context mutations

**Choice:** New `SignalTarget` as a sealed permit on `BindingTarget`
**Alternatives:**
- CapabilityTarget with a no-op signal worker — works today but papers over the abstraction gap; signals are not worker dispatches
- Extend `contextWrite` as a self-sufficient target — overloads pre-dispatch side-effect semantics
**Rationale:** The engine needs a first-class concept for "engine acts on its own state." Workers, sub-cases, and human tasks are all external actors. Timer-triggered SLA expiry, escalation flags, pathology signals — these are internal state transitions. `SignalTarget` fills a missing category in the `BindingTarget` hierarchy.
**Trade-offs:** Every exhaustive `switch` on `BindingTarget` needs a new branch — mechanical with IntelliJ, and the exhaustive switch is the safety mechanism that makes this safe.
**Exploration:** deep-analysis
**Status:** captured

## D2: Signal payload location

**Choice:** Payload on `SignalTarget(Map<String, Object> payload)` itself
**Alternatives:**
- Reuse `contextWrite` on Binding — conflates pre-dispatch side-effect with the primary action
**Rationale:** `contextWrite` is a pre-dispatch side-effect (fires before the target action). For signal bindings, the context write IS the action. Keeping it on the target preserves clean semantics and allows composition (a signal binding could also have `contextWrite` for pre-action setup).
**Trade-offs:** Two places can write to context on a single binding (`contextWrite` + `SignalTarget.payload`), but they have distinct semantics (setup vs action).
**Depends on:** D1 (SignalTarget exists)
**Exploration:** quick
**Status:** captured

## D3: YAML syntax for signal target

**Choice:** `signal:` block on binding, payload as nested map
**Alternatives:**
- `contextSignal:` — more explicit but verbose
- `action:` — too generic
**Rationale:** Reads naturally alongside `capability:`, `subCase:`, `humanTask:`. Consistent with existing target block naming convention.
**Trade-offs:** "signal" is overloaded in the engine (CaseHubRuntime.signal(), SignalType, typed signals), but within a binding definition the context is unambiguous.
**Depends on:** D1 (SignalTarget exists)
**Exploration:** quick
**Status:** captured

## D4: YAML ScheduleTrigger parity fix (bundled)

**Choice:** Implement `ScheduleTrigger` conversion in `CaseDefinitionYamlMapper.convertTrigger()` — fix the existing TODO at line 1116
**Alternatives:**
- None — this is a parity gap, not a design choice
**Rationale:** YAML and Java DSL must have parity. `ScheduleTrigger` exists in the Java model with `delay(Duration)` and `cron(String)` but the YAML mapper throws `UnsupportedOperationException`.
**Trade-offs:** None.
**Exploration:** quick
**Status:** captured

## D5: PlanItem lifecycle for SignalTarget

**Choice:** No PlanItem created for SignalTarget bindings
**Alternatives:**
- Create PlanItem, immediately mark COMPLETED — awkward state machine (DISPATCHING has no direct path to COMPLETED)
- New `markCompletedFromDispatching()` method — adds complexity for a transition that shouldn't exist
**Rationale:** Signals are instantaneous engine-internal actions, not work-in-progress. PlanItems track dispatched work; signals have no in-progress state. Audit trail comes from the EventLog entry (`CONTEXT_SIGNAL_APPLIED`). `LifecycleScope.BINDING` only — validated at build time in `Binding.Builder`, same pattern as `ScopeActivatedTrigger`.
**Trade-offs:** Signal bindings don't participate in compound completion counting. This is correct — they're side-effects, not tracked work items.
**Depends on:** D1 (SignalTarget exists)
**Exploration:** quick
**Status:** captured

## D6: Static payload only (v1)

**Choice:** `SignalTarget` payload is `Map<String, Object>` — literal values frozen at definition time
**Alternatives:**
- JQ expression payload — values evaluated against context at dispatch time
- Hybrid — literal by default, special syntax for expressions
**Rationale:** The SLA use case needs `{caseSla: {expired: true}}` — a boolean flag. Timestamps come from the EventLog entry, not the payload. Dynamic payloads (JQ-evaluated) are a backward-compatible addition when a concrete use case demands it. No need to design the expression syntax now.
**Trade-offs:** Cannot express derived values (elapsed time, context-dependent flags) in the payload. Acknowledged limitation for v1.
**Depends on:** D2 (payload on SignalTarget)
**Exploration:** quick
**Status:** captured
