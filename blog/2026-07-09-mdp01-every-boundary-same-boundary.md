---
layout: post
title: "Every Boundary Is the Same Boundary"
date: 2026-07-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [context-bridge, type-safety, java-generics]
series: issue-203-context-bridge-protocol
---

CaseHub passes data through five different boundary types — worker input, work items, sub-case context, signals, connectors. Every one of them uses `Map<String, Object>`. Every one of them has the same class of bugs: a key that should be `transaction` arrives as `txn`, a value that should be a `Double` arrives as an `Integer`, and the failure is silent because Maps don't know what they're supposed to contain.

We spent this session designing the fix — a single protocol called ContextBridge that applies the same typed translation at every boundary.

## The type capture problem

The core challenge is Java's type erasure. A worker function that wants to receive an `AmlTransaction` instead of a Map needs to declare that type somewhere the compiler can check it and the runtime can resolve it. `Class<T>` parameters work for simple types but can't express `Map<String, List<Transaction>>`. Serialized lambdas (the approach Serverless Workflow's FuncDSL used) can capture the type but depend on undocumented JVM serialization internals.

We landed on what we're calling the Reified Varargs Type Token. When a method declares `T... varargs`, Java creates a reified array at the call site even with zero arguments. `typeToken.getClass().getComponentType()` recovers the concrete class — `Map.class`, `AmlTransaction.class`, whatever `T` erases to. One declaration, zero allocation, compile-time safety on the lambda, runtime class for bridge resolution.

The DSL surface it produces:

```java
Worker.builder()
    .capabilityName("assess-risk")
    .<AmlTransaction>fn()
    .apply(txn -> WorkerResult.of(Map.of("risk", txn.riskScore())))
```

The `fn()` call captures the type. The `apply()` call binds the lambda. The compiler enforces that `txn` is `AmlTransaction`.

## Where the bridge sits

Each boundary type has the same structure: some source data needs to become a typed `T` for the consumer. The bridge handles that translation. For simple POJOs, Jackson converts automatically — the user never touches a bridge. For live execution contexts like `WorkflowContext` or `AgenticScope`, a bridge creates a scoped view backed by the `CaseContext`.

The critical design property is local scoping. In a five-level chain — Case → Flow → SubCase → Flow → SubCase — each level has its own `CaseContext` and its own bridge. No bridge reaches across levels. Parent data flows to children via input mapping (an existing mechanism), then the child's bridge reads from its own context. This means you can change a context type at any level without touching anything else.

We debated where the `ContextBridge<T>` SPI should live — `casehub-platform-api` (shared tier) or `casehub-engine-api` (engine tier). An adversarial debate with four advocates and a judge settled it: every module that actually calls `initialise()` already depends on engine-api. Placing it in platform-api would solve a dependency problem that doesn't exist.

## What the review caught

The spec went through two rounds of adversarial design review — 18 issues raised, all 18 verified and resolved. The significant ones: the `initialise()` signature originally took `String inputSchema`, but the engine's `JQEvaluator` lives in a different module tier than bridges — so the engine evaluates JQ first and passes a `JsonNode narrowedInput` to the bridge. And `PropagationContext`, which the original issue #203 specified as a bridge parameter, was moved to engine-level threading — snapshot bridges don't need it, and live-view bridges can access it through the pipeline.

The pattern that worked here was contributed back to the Serverless Workflow SDK — PR #1524 replaces their serialized lambda type inference with the Reified Varargs Type Token across ~40 call sites.
