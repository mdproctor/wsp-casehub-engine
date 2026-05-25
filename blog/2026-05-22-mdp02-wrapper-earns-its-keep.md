---
layout: post
title: "The Wrapper That Earns Its Keep"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [langchain4j, jq, api-design, refactoring]
---

Before touching any code I stopped and asked whether the casehub `Agent` class should exist at all. LangChain4j has its own agent model — `UntypedAgent`, `AgenticScope`, supervisor patterns. Was the casehub version just reinventing something the library already provides better?

The answer turned out to matter more than the refactoring.

The casehub `Agent` does one thing LangChain4j doesn't: it applies jq transformations to the case context *before* feeding data to the LLM, and to the LLM's output *after* getting it back. An `inputSchema: "{ title: .caseTitle, docs: .documents }"` extracts just what the model needs from the full case context. An `outputSchema: "{ decision: .verdict }"` writes only the relevant field back. LangChain4j handles the LLM call; casehub provides the bridge between its context model and whatever the LLM expects.

That's genuine value. The class stays — but it should be jq-agnostic. `Agent` was holding two `JqTransformer` fields, coupling it directly to a specific jq implementation. The real contract is a function: `UnaryOperator<JsonNode>`.

The change in `Agent` is one line per field. The change in `AgentBuilder` is more interesting: `inputSchema(String)` still works (it creates a `JqTransformer` internally and wraps `::apply`), but now `inputTransformer(UnaryOperator<JsonNode>)` gives CDI callers a way to pass a `JQEvaluator`-backed lambda. Defaults to identity when neither is set.

```java
final UnaryOperator<JsonNode> resolvedInput =
    inputSchema != null ? new JqTransformer(inputSchema)::apply
        : (inputTransformerFn != null ? inputTransformerFn : UnaryOperator.identity());
```

Writing the test for `AgentBuilder` hit a wall immediately. `ChatModel` can't be stubbed as a lambda — `"ChatModel is not a functional interface"`. Every method on it, including `chat(ChatRequest)` (the documented primary API), is `default`. Claude found `doChat(ChatRequest)` by reading the source: it's the real extension point, also `default`, throwing `RuntimeException("Not implemented")` unless overridden. The `chat` method orchestrates logging and delegates to `doChat`. Override `doChat`, not `chat`.

The second issue on this branch — `CommandContent` — was straightforward after the main work. `WorkerScheduleEventHandler.dispatchCommand()` was building its COMMAND content as a raw `HashMap` with string keys. Replacing it with a record makes the wire format visible:

```java
record CommandContent(
    String type, String capability, String correlationId,
    Map<String, Object> input, String deadline) {}
```

`@JsonInclude(NON_NULL)` on the record handles the optional deadline without branching. The wire format is unchanged — claudony reads JSON, not the Java type.

The bigger picture from this branch: the jq transformation layer is what makes AI workers composable with the case engine. The agent doesn't need to know what a case is. The case engine doesn't need to know how the model formats its output. The transformer between them is the seam.
