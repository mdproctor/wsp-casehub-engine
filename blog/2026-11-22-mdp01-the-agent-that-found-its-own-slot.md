---
title: "The Agent That Found Its Own Slot"
date: 2026-11-22
author: Mark Proctor
tags: [casehub, engine, eidos, agent, spi, yaml, design]
entry_type: note
subtype: diary
---

# The Agent That Found Its Own Slot

CaseHub's worker system has a clean extension SPI: `WorkerFunctionProvider` handles YAML parsing, `WorkerFunctionHandler` handles execution. Every worker type — flow, MCP, A2A, ReAct, pattern, playbook — goes through this SPI. Every type except one.

Agent workers were wired inline. The `YamlCaseDefinitionConverter` had a fallback path: if no provider handled the YAML node and an `agent:` block was present, it called `buildAgentFunction()` directly. The agent config — system prompt, model, projections — was baked into the YAML and resolved at parse time. No slot lookup, no eidos integration, no deferred resolution.

This matters because eidos agents have identity. They have descriptors with capabilities, dispositions, and system prompts rendered through `SystemPromptRenderer`. Hardcoding a system prompt in the YAML bypasses all of that — the YAML author is writing prompt engineering instead of referencing a managed agent definition.

## What Changed

I wanted two things: agents as first-class SPI participants, and a way for YAML to reference eidos agents by slot rather than inlining their config.

The core abstraction is `AgentSlotConfig` — a record holding a slot reference plus optional overrides (model, user message template, input/output projections). `AgentWorkerFunction` gains a dual-mode construction: eager (pre-built invoker, the existing path) and deferred (slot config, resolved at execution time). Two factory methods — `AgentWorkerFunction.eager(invoker)` and `AgentWorkerFunction.ofSlot(config)` — make the distinction explicit.

The YAML surface becomes:

```yaml
workers:
  - name: sentiment-analyst
    agent:
      slot: "sentiment-analyst"
      model: "claude-sonnet-5-5"
      inputProjection: ".ticket"
```

Where `slot` is the eidos agent's identity — the same value used in `AgentRegistry.find(AgentQuery.bySlot(...))`.

The resolution chain at execution time:

```
slot → AgentRegistry.find(bySlot)
     → AgentDescriptor
     → SystemPromptRenderer.render()
     → WorkerAgentInvoker
     → invocation
```

The agent's system prompt, capabilities, and disposition come from its registered descriptor. The YAML declares *which* agent, not *how* the agent works. Overrides (model, projections) are optional — they layer on top of the descriptor's defaults, not replace them.

## AgentTaskTarget

The second piece is `AgentTaskTarget` — a new sealed permit on `BindingTarget`. Where `CapabilityTarget` routes work to workers by capability name, and `HumanTaskTarget` (deprecated) routes to humans by candidate group, `AgentTaskTarget` routes to eidos agents by capability matching.

```java
Binding.builder()
    .name("analyse-ticket")
    .agentTask(AgentTaskTarget.builder()
        .capability("sentiment-analysis")
        .domain("customer-support")
        .build())
    .on(".ticket")
    .build();
```

At dispatch time, the resolver queries `AgentRegistry.find()` with the capability and domain, and the registry returns agents ordered by match quality. The default strategy picks the best match — the pool infrastructure is there for future selection strategies (round-robin, load-based) without changing the binding model.

Because `BindingTarget` is sealed, adding a new permit is a forcing function. The compiler finds every exhaustive switch — `BindingExecutorResolver`, `CaseDefinitionService`, `CompensationGraphProjection` — and refuses to compile until each handles the new case. Sealed interfaces earn their cost here.

## The Bug Claude Caught

During code review, Claude flagged a correctness bug in the handler. I'd written `agentRegistry.findById(slotConfig.slot(), tenancyId)` — but `findById` takes an `agentId`, not a slot. The two aren't the same field. The correct API is `AgentRegistry.find(AgentQuery.bySlot(slot, tenancyId))`, which queries by the slot field on `AgentDescriptor`. This would have silently failed for any agent where the slot and agentId differ — which is most of them in practice.

The fix was straightforward, but the bug was the kind that passes unit tests (the test stub was returning the descriptor regardless of which lookup method was called) and only surfaces in production when real descriptors have different IDs and slots. Code review caught it; the runtime wouldn't have.

## What This Opens Up

The helpdesk example in `casehubio/examples#212` can now define its case entirely in YAML — workers reference eidos agents by slot, bindings route by capability, and the case definition is a declarative composition of agent capabilities rather than a wiring diagram of system prompts and model configs.

The inline `agent:` path is gone. The converter's fallback is removed. `AgentWorkerFunctionProvider` owns the `agent:` key through the SPI, same as every other worker type. The `YamlAgent` generated record gained `slot`, `userMessage`, `inputProjection`, and `outputProjection` fields to support the new shape — a small change in the codegen mappings, but it's what makes the YAML→Java round-trip work through Jackson deserialization without losing the slot reference.
