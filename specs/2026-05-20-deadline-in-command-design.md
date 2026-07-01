# Deadline in COMMAND Content — Design

**Date:** 2026-05-20
**Issue:** casehubio/engine#300
**Status:** Approved

## Context

`WorkerScheduleEventHandler.dispatchCommand()` posts a COMMAND message to a worker
channel with a JSON content map. Claudony's `ClaudonyReactiveCaseChannelProvider`
consumes this content and uses it to open a Qhorus Commitment for obligation
tracking. To bound the Commitment's `expiresAt`, claudony needs the case budget
deadline — but it has no access to the engine's `CaseInstance` or
`PropagationContext`. The content JSON is the only channel.

`PropagationContext.getDeadline()` returns `Optional<Instant>`. It is present
when the case was started with a time budget; absent otherwise. The engine must
pass it through when present.

## Deferred / tracked

- **engine#301** — typed `CommandContent` record to replace the raw `Map`. The
  content map is a de facto protocol between engine and claudony; formalising
  it as a typed record would give compile-time schema enforcement. Deferred:
  adding one field is the right first step; schema formalisation follows.
- **engine#302** — align `CaseHub.startCase` / `CaseHubRuntime.startCase` with
  `Flow.instance(Object)`. Public entry points should accept `Object`, not
  `Map<String, Object>`, so callers can pass Maps, POJOs, or any
  JSON-serialisable type. Internal `inputData` parameters stay `Map<String,
  Object>` — they represent post-evaluation data from `inputMapping` expressions,
  not user-supplied input.

## Decision

Add `deadline` to the COMMAND content map when `PropagationContext` has a deadline.
No `CaseHub` API change this session — that waits for engine#302 to do it
correctly as `Object`.

## Design

### 1. `WorkerScheduleEventHandler.dispatchCommand()` (`runtime` module)

Switch from `Map.of(...)` (immutable) to `new HashMap<>()` so the deadline can
be conditionally added. Add the field via `ifPresent`:

```java
Map<String, Object> command = new HashMap<>();
command.put("type", "COMMAND");
command.put("capability", capability.getName());
command.put("correlationId", String.valueOf(eventLogId));
command.put("input", inputData);
instance.getPropagationContext().getDeadline()
    .ifPresent(d -> command.put("deadline", d.toString())); // ISO-8601
```

`Instant.toString()` produces ISO-8601 format (`2026-05-20T14:30:00Z`).
`deadline` is absent from the content when no budget was set.

### 2. Tests (`SpiWiringIntegrationTest`, `runtime` module)

Inject `CaseHubRuntime` directly alongside the existing `SimpleCaseHubBean`
to drive a case with a deadline without touching `CaseHub` API:

```java
@Inject CaseHubRuntime runtime;
```

**Test: deadline present**
```java
@Test
void commandContent_includesDeadline_whenCaseHasDeadline() {
    PropagationContext ctx = PropagationContext.createRoot(Map.of(), Duration.ofHours(1));
    runtime.startCase(simpleCaseHubBean.getDefinition(),
        Map.of("documentId", "doc-dl-1", "status", "processing"), null, ctx);

    await().atMost(15, SECONDS).untilAsserted(() ->
        assertThat(RecordingCaseChannelProvider.postedContents)
            .anyMatch(c -> c.contains("\"deadline\":")));
}
```

**Test: deadline absent**
```java
@Test
void commandContent_omitsDeadline_whenCaseHasNoDeadline() {
    simpleCaseHubBean.startCase(
        Map.of("documentId", "doc-nodl-1", "status", "processing")).join();

    await().atMost(15, SECONDS).untilAsserted(() ->
        assertThat(RecordingCaseChannelProvider.postedContents)
            .anyMatch(c -> c.contains("\"type\":\"COMMAND\"")));

    assertThat(RecordingCaseChannelProvider.postedContents)
        .filteredOn(c -> c.contains("\"type\":\"COMMAND\""))
        .noneMatch(c -> c.contains("\"deadline\""));
}
```

### 3. CLAUDE.md — SPI wiring table

Update the `dispatchCommand` row to document the COMMAND content fields:

```
| CaseChannelProvider.openChannel + postToChannel(..., MessageType.COMMAND)
| WorkerScheduleEventHandler.dispatchCommand
| Worker scheduled — opens channel, posts COMMAND. Content fields: type,
  capability, correlationId, input, deadline (optional ISO-8601 Instant —
  present when PropagationContext has a deadline, absent otherwise) |
```

### 4. Documentation — Map vs Object convention

**Javadoc** on `io.casehub.engine.internal.worker.WorkflowExecutor#execute` and
`io.casehub.engine.spi.scheduler.WorkerExecutionManager#submit`:

> `inputData` is `Map<String, Object>` at this layer because it represents
> post-evaluation data — the result of applying `inputMapping` expressions
> against `CaseContext`. This is the correct type at the engine-internal layer.
> Public entry points (`CaseHub.startCase`, `CaseHubRuntime.startCase`) should
> accept `Object` to align with `Flow.instance(Object)` — tracked in engine#302.

**Protocol** in `docs/protocols/casehub/`:

Capture the rule: internal `inputData` is `Map<String, Object>` (correct —
post-deserialization); public entry points should be `Object` (pending #302).
Prevents future reviewers asking the same question.

## What is not changed

- `CaseHub.startCase` — no new overload. Waits for engine#302.
- `CaseHubRuntime` — no new overload.
- All internal `Map<String, Object> inputData` parameters in handlers,
  executors, and schedulers — correct at that layer, no change.
- COMMAND content schema type — stays raw `Map`. engine#301 formalises it.