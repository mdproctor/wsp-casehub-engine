---
layout: post
title: "Every Resolution, Same Pipeline"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [cbr, retrieval, human-in-the-loop, judgment, feedback, resolution]
series: issue-1081-unified-resolution-pipeline
---

# Every Resolution, Same Pipeline

CaseHub's CBR retrieval works for the fully automated case: retrieve similar past cases, score agents by historical success, dispatch the best fit. But real case management has four quadrants. An agent can select and an agent can execute. A human can select and an agent can execute. An agent can select and a human follows the procedure manually. A human can pick and a human can follow. The engine only served the first quadrant.

The gap isn't just "add a human option." It's that knowledge base articles — runbooks, SOPs, investigation procedures — can't participate in retrieval at all. `ResolutionGuide` exists in neocortex as a CBR case type, but the engine never ingests documents and never presents retrieval results to a human for selection. And when retrieval does inform a dispatch, nobody records whether the retrieved cases were actually relevant. The feedback loop is open.

I wanted to close all of these at once because they're coupled. Mixed retrieval (plan traces alongside documents) is pointless without a way to present the mixed results. Presenting results to a human is pointless without feeding the selection back. And feedback is noise without outcome weighting to amplify it.

## The design that fell out

The key insight was that this isn't a new module. Every piece already has a home.

`CbrRetrievalService` handles retrieval — extending it for mixed results means mapping `ResolutionGuide` to `RetrievedExperience` alongside the existing `ResolvedCase` mapping. The neocortex CBR store already supports dense vector search, SPLADE, BM25, and feature similarity — whole-document retrieval through CBR is functionally equivalent to RAG, with the addition of structural feature matching that RAG lacks.

For human selection, `JudgmentTarget` already handles human-in-the-loop decisions with structured outcomes, verification, and escalation. Resolution candidates are a richer payload, not a different mechanism. The pattern: a judgment binding fires before the capability binding, presents ranked candidates, the human selects, the selection writes to context, and the capability binding dispatches. For the automated path, the judgment binding is absent — no ceremony.

For feedback, `StepOutcomeObserver` is the right hook. It fires after each worker step with the outcome. A `RetrievalFeedbackObserver` reads the retrieval trace from EventLog metadata, maps the outcome to relevance, and calls `CbrRetrievalTracker.feedback()`. Three feedback layers: per-step retrieval relevance, per-case outcome (already wired via `CbrCaseRetainObserver`), and per-judgment selection feedback.

## What the spec review caught

Claude found that `ResolutionStep` collides with an existing neocortex type — the plan execution trace step uses that name. Renamed to `GuidanceStep`. It also caught that `StepOutcomeObserver` uses single-dispatch (`.get()`) instead of iterating — adding a second observer would cause `AmbiguousResolutionException`. And that `handleSemanticFailure()` hardcodes `RoutingOutcome.FAILURE` for declined outcomes, making the DECLINED-specific feedback mapping unreachable. Both are prerequisite fixes in the handler.

The cross-type retrieval path had an untested assumption: `retrieve()` passes a resolved `caseClass` parameter that constrains the generic return type. When `crossType: true`, the class must be `CbrCase.class` (the interface), not the `cbrType`-resolved concrete class — otherwise `ResolutionGuide` results can't coexist with `ResolvedCase` results in the same container.

## What landed

Foundation types are in. `ResolutionSourceType` discriminates plan traces from resolution guides. `DocumentStep` maps from neocortex's `GuidanceStep`. `ResolutionSelection` carries the human's choice with optional rationale. `RetrievedExperience` gained three fields — `sourceType`, `documentContent`, `documentSteps` — with backward-compatible constructors.

The CBR naming cleanup touched twelve files: `PlanCbrCase` to `ResolvedCase`, `TextualCbrCase` to `ResolutionGuide`, `PlanTrace` to `ResolutionStep`, and the `planTrace()` accessor to `resolutionStep()` on `ResolvedCase`. The neocortex rename landed months ago; the engine was still importing the old names.

## What's next

Five batches remain, gated on two neocortex changes: `GuidanceStep` plus `features` field on `ResolutionGuide`, and a `feedback()` method on `CbrRetrievalTracker`. Once those land, mixed retrieval mapping, the ingestion SPI, feedback wiring, and the judgment candidate presentation can proceed. The candidate presentation is the most complex piece — writing candidate summaries to context while suppressing `CONTEXT_CHANGED` to prevent circular dispatch.
