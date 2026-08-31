---
layout: post
title: "The Deserializers That Deserialised Themselves"
date: 2026-08-31
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [yaml, jackson, records, refactoring]
---

# The Deserializers That Deserialised Themselves

The engine's YAML loading pipeline had accumulated 2,500 lines of hand-coded Jackson deserializers. `CaseDefinitionDeserializer` alone was 793 lines — fifty-odd fields wired with `if (node.has("X")) def.setX(...)` boilerplate. `BindingDeserializer` added another 670. A post-processor added 472 more. And between them sat `CaseDefinitionSpec`, a 408-line intermediate model that existed only so Jackson could populate fields that `CaseDefinition` would later delegate to it.

The thing about boilerplate deserializers is that they're individually correct and collectively terrible. Each `if` block works. The field mapping is right. But the whole structure fights Jackson instead of using it. Jackson already knows how to map YAML fields to record components — we were doing its job manually, across four files, and maintaining each one by hand whenever a new field arrived.

The replacement is plain Java records. Thirty-two small records that mirror the YAML shape exactly: `YamlCaseDefinition`, `YamlCaseSpec`, `YamlWorker`, `YamlBinding`, and twenty-eight supporting types. Jackson deserialises to them with zero custom code. Polymorphic types — `Trigger`, `CaseCompletion`, `ExpressionEvaluator` — keep their existing deserializers via `@JsonDeserialize` annotations on record fields. Everything else is auto-mapped.

A single converter class, `YamlCaseDefinitionConverter`, handles the domain transforms: parsing `Path.parse()` for types and labels, `Class.forName()` for signal types and context bridges, baking JQ expressions, constructing GOAP actions from shorthand, resolving worker functions through the provider registry. This is the part that was genuinely hand-written before and genuinely needs to be — it's domain logic, not field mapping. The difference is that it's one class with one responsibility, not logic scattered across four files interleaved with boilerplate.

With the records and converter in place, I inlined `CaseDefinitionSpec` into `CaseDefinition`. The spec was an unnecessary layer — every getter on `CaseDefinition` delegated to `spec.getX()`, every setter called `spec.setX()`. Thirty fields, sixty methods, all doing nothing but forwarding. After inlining, `CaseDefinition` owns its fields directly and `CaseDefinitionSpec` is gone.

The net deletion was around 3,200 lines across twelve files — three deserializers, a post-processor, an intermediate model, three mixins, and four test classes that tested the dead code. The api module went from 1391 tests to 1373 (the deleted tests tested the deleted path; equivalent coverage exists in the mapper tests).

The second half of the work wired yaml-core's `ForEachExpander` and `VariableResolver` into the loading pipeline. Case definitions can now declare `iterations:` blocks and use `forEach:` on workers and bindings for template expansion — `processor-${each.region}` with `regions: [eu, us, ap]` expands to three workers. Variable resolution supports `${env.X}` and `${config.X}` with deferred `${each.*}` references that resolve during expansion. This required a one-line fix in yaml-core's `VariableResolver` — the `each` prefix was hardcoded before the deferred-prefix check, making two-phase resolution impossible.

The records are the interesting design choice here. Each one is a pure data carrier with a compact constructor for null-safe defaults. No validation, no transformation, no business logic. The YAML shape is the shape — if the YAML has a `forEach` field on a worker, `YamlWorker` has a `String forEach` component. Jackson does the rest. The converter is where shape meets semantics.
