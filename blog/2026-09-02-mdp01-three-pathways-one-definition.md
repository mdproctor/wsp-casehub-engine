---
layout: post
title: "Three Pathways, One Definition"
date: 2026-09-02
entry_type: article
subtype: diary
projects: [casehubio/engine]
tags: [yaml, dsl, annotations, case-definition, rosetta-stone, examples]
---

# Three Pathways, One Definition

CaseHub has three ways to define a case: YAML, a Java DSL, and Java annotations. Until now, there was no place where the same definition existed in all three forms — so the question "how does this YAML feature look in the DSL?" had no concrete answer. I built eight Rosetta Stone example sets to fix that, and the exercise exposed where each pathway genuinely shines and where it hits a wall.

## The Concept

A Rosetta Stone example is the same case definition expressed three ways. Same namespace, same capabilities, same workers, same bindings, same goals. Three files. A developer picks the pathway they know and reads across to the one they don't.

The eight execution models I covered, each with a different realistic domain:

| Model | Domain | What it demonstrates |
|---|---|---|
| Choreography | Banking — customer onboarding | The default: bindings fire independently on context changes |
| Sequential | HR — employee onboarding | `planningStrategy: sequential` — one binding at a time |
| GOAP | Legal — contract review | Planner computes order from preconditions and effects |
| LLM decomposition | Research — analysis pipeline | LLM breaks a goal into steps from available capabilities |
| HumanTask | Finance — loan approval | WorkItems routed to human inboxes with outcomes and SLA |
| SubCase | Insurance — claims processing | Parent case delegates investigation to a child case |
| A2A | Intelligence — market research | Remote agents via the A2A protocol |
| MCP | DevOps — code analysis | MCP server tools for static analysis and vulnerability scanning |

## How the Mapping Works — Choreography

The simplest case shows the pattern. A bank opens a new customer account. Three capabilities fire independently as context changes: verify identity, run KYC screening, provision the account.

### YAML

YAML is the most readable pathway. Everything is declarative — the structure maps directly to the conceptual model:

```yaml
spec:
  capabilities:
    - name: verifyIdentity
      inputProjection: "{ application: .application }"
      outputProjection: "{ identityResult: { verified: .verified, referenceId: .referenceId } }"
    - name: kycScreening
      inputProjection: "{ identityResult: .identityResult }"
      outputProjection: "{ complianceResult: { status: .status, referenceId: .referenceId } }"

  bindings:
    - name: verify-on-application
      capability: verifyIdentity
      on:
        contextChange:
          filter: '.application != null and .identityResult == null'
    - name: screen-after-verified
      capability: kycScreening
      on:
        contextChange:
          filter: '.identityResult != null and .complianceResult == null'
      when: '.identityResult.verified == true'
```

Capabilities declare what goes in and what comes out. Bindings declare when each fires. The `when:` guard adds a second condition checked after the trigger. A reader who has never seen CaseHub can follow this.

### DSL

The Java DSL expresses exactly the same thing with builder APIs. It's more verbose, but the type system catches errors that YAML silently accepts:

```java
Capability verifyIdentity = Capability.of(
    "verifyIdentity",
    "{ application: .application }",
    "{ identityResult: { verified: .verified, referenceId: .referenceId } }");

return CaseDefinition.builder()
    .namespace("banking")
    .name("customer-onboarding")
    .capabilities(verifyIdentity, kycScreening, provisionAccount)
    .bindings(
        Binding.builder()
            .name("verify-on-application")
            .capability(verifyIdentity)
            .on(new ContextChangeTrigger(
                ".application != null and .identityResult == null"))
            .build(),
        Binding.builder()
            .name("screen-after-verified")
            .capability(kycScreening)
            .on(new ContextChangeTrigger(
                ".identityResult != null and .complianceResult == null"))
            .when(".identityResult.verified == true")
            .build())
    // ...
    .build();
```

The binding references `verifyIdentity` as a Java variable — the compiler verifies the capability exists. In YAML, `capability: verifyIdenity` (note the typo) would parse fine and fail at runtime.

### Annotations

The annotation pathway is the most concise. Worker methods are the case definition — parameters become preconditions, return types become effects, and `@Bind` declares when:

```java
@Case(namespace = "banking", name = "CustomerOnboarding", version = "1.0.0")
public interface SimpleAnnotatedCase {

  @Worker(capability = "verifyIdentity")
  @Bind(contextChange = ".application != null")
  default IdentityResult verifyIdentity(String application) {
    return new IdentityResult(true, "ID-" + application.hashCode());
  }

  @Worker(value = "kycScreening")
  @Bind(contextChange = ".identityResult != null",
        when = ".identityResult.verified == true")
  @Bind(cron = "0 0 * * * ?")
  default ComplianceResult checkCompliance(IdentityResult identityResult) {
    return new ComplianceResult("PASS", identityResult.referenceId());
  }
}
```

Three lines of annotation replace eight lines of builder code. The worker function IS the definition — the method signature tells you input and output types, and the Quarkus build step generates the `CaseDefinition` bean at compile time.

## Where Each Pathway Shines

**YAML** is the right choice when the case definition is configuration, not code. No JVM, no compilation step, no Maven project. A domain expert can read and modify it. Workers reference external services via `do:` blocks (Serverless Workflow HTTP calls), `a2a:` blocks (remote agents), or `mcp:` blocks (MCP tool servers). The YAML pathway is the only one where the entire definition lives in one file.

**DSL** is the right choice when you need programmatic control. Conditional logic, dynamic capability registration, shared constants across definitions, computed input projections — anything that benefits from being code. The type system catches entire categories of errors at compile time. And the DSL has access to every API the engine offers — nothing requires an escape hatch.

**Annotations** are the right choice for rapid prototyping and simple cases. A `@Case` interface is the fastest way to get a running case definition — method signatures infer GOAP preconditions and effects automatically, `@Worker` methods are both definition and implementation, and the whole thing fits in one class. For choreography cases with in-process workers, annotations are hard to beat.

## Where Each Pathway Hits a Wall

This is where the Rosetta Stone exercise proved its value. Building all eight models in all three pathways exposed the annotation boundaries precisely.

### Annotations can't express deployment-time configuration

`a2a:` and `mcp:` blocks on workers are deployment-time configuration — which remote endpoint to call, what authentication to use, whether to stream. There's no annotation for this because there shouldn't be: the endpoint URL isn't a property of the case definition, it's a property of the deployment.

The annotated A2A and MCP examples use placeholder in-process workers. The case structure is identical, but the workers don't call remote services. The YAML examples are the canonical reference for these execution models.

### Annotations need `@Customize` for engine-level features

Several engine features live outside what annotations can infer from method signatures:

```java
@Customize
static void customize(CaseDefinition.Builder builder) {
  builder
      .decompositionStrategy("llm")
      .planningConstraints(PlanningConstraints.of(Duration.ofHours(1), 3))
      .adaptationConfig(AdaptationConfig.of("every-step", "forward-replan"));
}
```

`decompositionStrategy`, `planningStrategy`, `planningConstraints`, `adaptationConfig` — these are case-level configuration with no natural mapping to method annotations. The `@Customize` block drops into the DSL to fill the gap, and it's visible in the source code as documentation of exactly where annotations reach their limit.

### HumanTask and SubCase bindings require `@Customize`

The most significant annotation limitation: `@Bind` can only target capabilities. HumanTask bindings (which create WorkItems in a human inbox) and SubCase bindings (which spawn child cases) require the full DSL:

```java
@Customize
static void customize(CaseDefinition.Builder builder) {
  builder.binding(
      Binding.builder()
          .name("officer-approval")
          .judgment(
              JudgmentTarget.builder()
                  .prompt("Review loan application")
                  .outcomes(Set.of("APPROVED", "DECLINED", "REFERRED"))
                  .expiresIn(Duration.ofHours(48))
                  .human(new HumanRoutingConfig(
                      null,
                      new CandidateSetSpec.Inline(
                          StaticSetStrategy.of("loan-officers", "senior-underwriters")),
                      null, 4, null))
                  .build())
          .on(new ContextChangeTrigger(
              ".riskAssessment != null and .decision == null"))
          .build());
}
```

The `@Customize` block for the loan approval HumanTask example is longer than the two `@Worker` methods combined. At this point, the annotation pathway is the DSL with extra steps.

### GOAP: where annotations genuinely add value

The GOAP example is where annotations are at their strongest. The planner needs preconditions, effects, costs, and soft dependencies — and annotations infer all of these from the method signature:

```java
@Worker(capability = "assessRisk", cost = 0.5)
@Effect("riskAssessment")
default RiskReport assessRisk(
    AnalysisResult analysisResult,
    ClauseList clauseList,
    @SoftDependency LegalOpinion legalOpinion,
    @SoftDependency PriorReview priorReview,
    @Param("jurisdiction") String jurisdiction) {
  // ...
}
```

Parameters become hard preconditions. `@SoftDependency` parameters become soft preconditions — the planner prefers them but proceeds without. `@Effect` overrides the default effect key. `@Param` marks a parameter that isn't a precondition at all. The YAML equivalent requires a separate `actions:` block with explicit maps:

```yaml
actions:
  - name: assessRisk
    preconditions:
      analysisResult: true
      clauseList: true
    effects:
      riskAssessment: true
    cost: 0.5
    softPreconditions:
      legalOpinion: true
      priorReview: true
```

The annotation version is both shorter and safer — the compiler verifies that `AnalysisResult` actually exists as a type in the project. The YAML version is a bag of strings that could reference anything.

## The Escape Hatch Inventory

Building all eight models produced a concrete inventory of what annotations can and cannot express:

| Feature | YAML | DSL | Annotations |
|---|---|---|---|
| Choreography bindings | native | native | native |
| Sequential planning | native | native | `@Customize` |
| GOAP actions | native | native | inferred from signatures |
| LLM decomposition | native | native | `@Customize` |
| HumanTask targets | native | native | `@Customize` |
| SubCase targets | native | native | `@Customize` |
| A2A/MCP workers | native | `noFunction()` | placeholder workers |
| Agent model config | native | native | `@Customize` |
| Planning constraints | native | native | `@Customize` |
| Adaptation config | native | native | `@Customize` |

The pattern is clear. Annotations handle workers and their behavioural metadata well — capabilities, costs, effects, dependencies. They don't handle case-level configuration or non-capability binding targets. The `@Customize` escape hatch exists precisely for this boundary, and these examples document every place it's needed.

## A Fix Along the Way

Claude caught a subtle YAML bug during the robustness audit: the `outputMapping` on the HumanTask example was at binding level instead of inside the `humanTask:` block. The schema defines `outputMapping` as a property of `HumanTask`, not of `Binding` — so at binding level it would be silently ignored at runtime. The schema validator didn't catch it because `unevaluatedProperties` interacts badly with `oneOf` in the networknt validator (there's even a disabled test for this exact limitation). The YAML passed validation but would have produced wrong behaviour.

This is exactly the kind of bug these examples are meant to prevent. A developer copying the pattern would have inherited the mistake.

## What This Opens Up

The eight examples are the foundation for a Three Pathways Guide — a documentation page that links the three forms and explains when to choose each. The remaining 15 execution models from the parent epic are follow-on work: patterns like composable routing, contingent planning, lifecycle scopes, and the ReAct execution loop. Each will follow the same Rosetta Stone structure.

The examples also serve as diagram test fixtures for blocks-ui — each YAML file is a valid case definition that the orchestration workbench can render. That's a lot of free test coverage for a documentation exercise.
