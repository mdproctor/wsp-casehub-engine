---
layout: post
title: "Examples as gap detectors"
date: 2026-09-15
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [cbr, annotations, parity, examples]
---

I wanted to build a tutorial example showing CBR ensemble consensus end-to-end — agents reading `.cbrEnsemble.stepAnalysis` to see which past response steps have UNANIMOUS agreement versus CONTESTED disagreement. Three pathways: YAML, Java DSL, and annotations. The kind of example a practitioner copies as a starting point.

The YAML and DSL paths worked immediately. The annotation path didn't — there was no `@Cbr` annotation. And looking closer, the YAML JSON Schema was missing two fields (`crossType` and `problemDescription`) that the deserializer already handled at runtime. The Java API had them, the runtime parsed them, but the schema didn't validate them and the annotation layer didn't know they existed.

Three gaps, all the same class of bug: a feature added to one pathway without propagating to the others. The deserializer gap was particularly quiet — YAML with `crossType: true` worked fine at runtime because `CbrConfigDeserializer` parsed it, but schema validation couldn't catch a typo because the field wasn't declared. A user writing `crosstype: true` (lowercase t) would get silently ignored rather than a validation error.

We fixed all three: added the missing schema fields, built the full `@Cbr` annotation pipeline (annotation types, Jandex scanning in the deployment processor, `CbrConfig` mapping in the recorder), and wrote the three-pathway example. The annotation work followed the established pattern — `@Feature(name, expression)` and `@Weight(name, value)` as nested annotations since Java annotations can't have Map fields, sentinel `-1` for optional integers since they can't be null.

The example itself uses an incident triage scenario: five past incident responses are retrieved via CBR, the `PlanEnsembleAnalyzer` classifies step-level agreement, and the triage agent reads the consensus to identify which response steps have broad agreement and which are contested. Same scenario in all three pathways — the point is showing the same capability expressed in each representation.

Along the way, we also added `retrieveForSelectionWithEnsemble()` to `CbrRetrievalService` — the existing `retrieveForSelection()` returned raw `List<RetrievedExperience>` with no ensemble analysis. The new overloads take a `caseType` parameter and return `CbrRetrievalResult` with ensemble consensus. No adaptation (no `CaseDefinition` available in the selection path), so raw plan traces are wrapped as RETAINED `AdaptedPlan` — same fallback pattern the main retrieval path uses for partial adaptation failures.

The pattern that emerged is worth naming: examples are gap detectors. The act of showing a feature across all three pathways forces you to verify the feature actually works in all three. We formalised this as a protocol — API surface parity across YAML, DSL, and annotations. Gaps between pathways are bugs. Every new `CaseDefinition` field should be available in all three representations, and building an example that uses it across all three is the verification.
