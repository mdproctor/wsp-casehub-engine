---
title: Three Bugs Hiding Behind Each Other
date: 2026-11-23
author: mdp
entry_type: note
subtype: diary
projects:
  - casehubio/engine
tags:
  - cdi
  - yaml
  - jackson
  - debugging
series: issue-1339-agent-task-dispatch
---

# Three Bugs Hiding Behind Each Other

The casehub-engine has a provider pattern for YAML worker types. Each worker kind — `do:` blocks, `agent:` slots, `a2a:` endpoints, `mcp:` servers — gets a `WorkerFunctionProvider` that recognises its YAML structure and produces the right `WorkerFunction` subtype. The converter iterates all registered providers until one handles the node.

The `FlowWorkerFunctionProvider` for `do:` block workers existed since day one. It had tests. It compiled. It passed code review. And it had never worked.

## Bug one: no CDI annotation

`FlowWorkerFunctionProvider` implements `WorkerFunctionProvider` but had no `@ApplicationScoped` annotation. The CDI container never discovered it. The `DefaultWorkerFunctionProviderRegistry` injects `Instance<WorkerFunctionProvider>` and collects all CDI beans — an unannotated class is invisible.

The fix is one line:

```java
@ApplicationScoped
public class FlowWorkerFunctionProvider implements WorkerFunctionProvider {
```

But adding this line revealed the second bug.

## Bug two: `do` versus `doBlock`

`do` is a Java keyword. The codegen generates YAML deserialization records from a mapping file, and when a YAML field name collides with a keyword, it renames the Java field:

```yaml
# yaml-record-mappings.yaml
Worker:
  fields:
    do: { name: doBlock, type: JsonNode }
```

This produces a record component named `doBlock`. Without `@JsonProperty("do")`, Jackson deserialises the YAML `do:` field into... nothing. The annotation wasn't generated because the codegen didn't auto-emit `@JsonProperty` for renamed fields. The `doBlock` component was always null.

The converter calls `MAPPER.valueToTree(yamlWorker)` to re-serialise the record for provider matching. With `doBlock` as the field name and no `@JsonProperty`, the JSON output has `"doBlock": null` — the original `do:` array is lost.

The codegen fix is adding `property: do` to the mapping:

```yaml
do: { name: doBlock, type: JsonNode, property: do }
```

This generates `@JsonProperty("do") JsonNode doBlock` on the record component. Jackson now correctly round-trips the field: deserialise `do:` → `doBlock`, serialise `doBlock` → `"do"`.

Adding this fix revealed the third bug.

## Bug three: null matches everything

With `@JsonProperty("do")` on `doBlock`, the re-serialised JSON for *every* worker now includes `"do": null`. Workers that don't have a do-block still get the field — it's just null.

`FlowWorkerFunctionProvider.handles()` checked `rawWorkerNode.has("do")`. Jackson's `has()` returns true for null values. Every worker matched.

The `create()` method then tried to parse the null value as a Serverless Workflow definition: `/do: null found, array expected`. This crashed the entire Quarkus startup — not just the do-block workers, but every `YamlCaseHub` bean in the application.

The fix:

```java
public boolean handles(final JsonNode rawWorkerNode) {
    return rawWorkerNode.has("do") && !rawWorkerNode.get("do").isNull();
}
```

## Why all three were invisible

Each bug was invisible because the previous bug in the chain prevented it from manifesting:

```
Bug 1: No @ApplicationScoped  → provider never discovered
                                 ↓ (fix reveals)
Bug 2: No @JsonProperty("do") → doBlock always null, do: array lost
                                 ↓ (fix reveals)
Bug 3: null matches has("do") → all workers match, startup crashes
```

Bug 1 masked bug 2 because the provider never ran. Bug 2 masked bug 3 because without `@JsonProperty`, the serialised JSON had `"doBlock": null` (not `"do": null`), so `has("do")` returned false. Only when both bugs 1 and 2 were fixed simultaneously did bug 3 surface — a null value on a field that now exists in every serialised worker.

The existing unit tests for `FlowWorkerFunctionProvider` passed because they constructed the provider directly with `new FlowWorkerFunctionProvider()` and fed it hand-built JSON nodes with the correct `"do"` key. They bypassed CDI, bypassed the YamlWorker round-trip serialisation, and tested only the happy path where the input was already correct.

## The integration test that exposed it

The trigger was issue #1336 — write a `@QuarkusTest` that verifies YAML do-block workers are properly wired through the CDI pipeline. The test loaded `simple.yaml` (which has a `do:` block worker), injected `WorkerFunctionProviderRegistry`, and checked that the parsed worker function is a `FlowWorkerFunction`.

Without this test, the three bugs would have continued hiding behind each other. The unit tests were green. The YAML file loaded without errors (workers silently got `WorkerFunction.NONE`). The existing programmatic `CaseHub` beans bypassed YAML entirely. No observable failure anywhere.

The moral: if a wiring path has no integration test exercising the full CDI→serialise→match→construct chain, it doesn't work. It just looks like it does.
