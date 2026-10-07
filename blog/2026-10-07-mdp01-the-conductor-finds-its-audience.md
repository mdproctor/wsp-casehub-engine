---
layout: post
title: "The Conductor Finds Its Audience"
date: 2026-10-07
entry_type: note
subtype: diary
projects: [casehubio/engine, casehubio/devtown]
tags: [evolution-conductor, capability-areas, devtown, first-consumer]
series: issue-1180-evolution-conductor-ui-in-devtown
---

# The Conductor Finds Its Audience

The evolution conductor has been built for months — five layers of orchestration (observe, summarize, control, steer, review), pluggable SPIs from the #1148 generalisation, blocks-ui workbench components with deny/watch/gate editors and an inbox. All tested. All rendering against mock data.

Nobody uses it yet.

Devtown changes that. It's the first application that has both the data surface and the motivation. A PR review coordination harness that routes work to specialist agents, tracks reviewer trust, manages merge queues, and enforces SLAs — that's a system with enough moving parts to benefit from a conductor watching its health and proposing improvements.

## What the conductor needs from a consumer

The conductor is domain-agnostic after #1148. It doesn't know what "healthy" means — each domain defines that through `CapabilityArea` implementations. It doesn't know what improvements are possible — each domain defines that through `ImprovementCategoryProvider`. The orchestration backbone (ticker, circuit breaker, budget enforcer, regression detection) just runs the loop.

So the design question was: what does "healthy" mean for devtown, and what improvements can the conductor actually drive?

## Five sensors, one pipeline

I landed on five capability areas that trace devtown's PR lifecycle end-to-end:

**CI Reliability** (weight 0.25) — build pass rate. If CI is broken, nothing else matters. This is the foundation score.

**Review Quality** (weight 0.25) — the ratio of accepted findings to total findings. Are the LLM reviewers producing signal or noise? When accuracy drifts, the conductor should notice before anyone reads a false-positive review comment.

**Merge Queue Health** (weight 0.20) — throughput, batch success rate, time-to-merge. The queue is where work lands or stalls.

**Reviewer Trust** (weight 0.15) — are trust scores calibrated? Do high-trust reviewers actually produce better outcomes? Is work evenly distributed?

**SLA Compliance** (weight 0.15) — human task gates completing within SLA. Escalation rate. Triage backlog depth.

These five cover the full lifecycle: code arrives → gets reviewed → trust routes it → human gates fire → it merges. The weights reflect where degradation hurts most.

## Improvements that aren't code changes

The interesting design choice was the improvement categories. The engine's default categories are code-level: dependency updates, lint fixes, coverage gaps. Devtown's categories are configuration-level: reviewer calibration, routing adjustment, SLA tuning, gate tightening, queue optimisation.

This matters because the improvement pipeline is shorter. Code changes need eleven stages (introspect through outcome-recording). Configuration changes need five: analyze, propose, review, apply, observe. Both `propose` and `review` are gate checkpoints — every devtown improvement requires explicit conductor approval until trust builds.

The conductor isn't fixing devtown's source code. It's tuning devtown's orchestration. When reviewer trust scores drift, the conductor proposes recalibrating the weights. When merge queue throughput drops, it proposes adjusting batch sizes. When SLA completion rates fall, it proposes looser thresholds.

This is what makes devtown genuinely the "first consumer" rather than just another codebase running the generic code-evolution loop. The conductor improves the system it's embedded in — and the feedback is visible in the same dashboard that shows the orchestration it's improving.

## What's built, what's next

The five capability areas and the category provider are implemented and tested — 22 unit tests, all green. Each area follows the same pattern as the engine's existing ten: extend `AbstractCapabilityArea`, read from `EventLogRepository`, compute a health score, return an assessment.

Three batches remain. The filtering SPIs (conflict detection, deny patterns, regression evaluation) define the safety rails. The API facade wires the engine's evolution API through devtown with enrichment — improvement streams that show PR context, not raw IDs. And the frontend adds a ninth dashboard tab instantiating the blocks-ui evolution workbench.

The workbench components are already built. The conductor is already built. The design decisions are captured. What's left is plumbing — and the moment devtown's dashboard shows a real health score computed from its own PR pipeline data, the feedback loop closes.
