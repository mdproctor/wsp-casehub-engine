---
layout: post
title: "Two gates before you observe"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [compliance, trust, eidos, behavioral-observation]
---

The engine already observes two compliance dimensions — latency and attestation rate — and feeds them to eidos for trust scoring. Both are straightforward: measure the clock, check the outcome. Adding the two remaining dimensions, delegation and escalation, turned out to be a different kind of problem.

Latency has a natural gate: if the capability doesn't declare a `latencyHintP50Ms`, there's nothing to measure. No hint, no observation. Attestation has no gate at all — every outcome is a data point. But delegation and escalation are policy judgments, not measurements. The question isn't "how long did it take" but "did the agent behave appropriately given what it was supposed to do?"

We landed on a two-gate model. Gate 1 is the eidos disposition — does this agent's profile say it should delegate (or should be supervised for escalation)? Gate 2 is engine-side: is delegation structurally possible for this task? For delegation, that means checking whether the `CaseDefinition` has decomposition infrastructure — a `decompositionStrategy` or `SubCaseTarget` bindings. No infrastructure, nothing to delegate to, no observation. For escalation, the eidos gate alone is narrow enough — only agents whose autonomy vocabulary term `impliesSupervision()` are observed.

The interesting design tension was around VIOLATED signals. The first cut said escalation could only record COMPLIANT — we could detect when an agent *did* escalate (returned a `PlannedAction` or declined the task), but couldn't detect when it *should have* escalated but didn't. A decision review caught this: if the eidos gate already narrows to supervised agents only, then a supervised agent completing autonomously without flagging anything *is* a meaningful negative signal. The "too noisy" concern melts away when you realise how few agents pass the supervision gate in practice. We revised to record VIOLATED on autonomous success — same treatment as delegation, where structural availability without evidence of delegation is also VIOLATED.

The cross-dimension interaction is worth naming explicitly. A `Declined` outcome is VIOLATED for attestation (the agent didn't do the work) and COMPLIANT for escalation (the agent recognised its limits). Same event, opposite signals on different dimensions. The trust model consumes both — deployments weight them differently depending on whether reliability or safety is the priority.

The implementation itself was mechanical — two private methods in `BehavioralComplianceRecorder` following the exact pattern of the existing `recordLatency` and `recordAttestation`. Two new constructor dependencies: `Instance<PlanItemStore>` for delegation evidence (checking compound children) and `VocabularyRegistry` for escalation eligibility. The delegation evidence is case-level, not per-execution — a known v1 limitation where pre-decomposed cases give COMPLIANT signals to agents that didn't personally delegate. Per-execution tracking via `WorkerRuntime` flags is the obvious v2 path when it matters enough.

The CDI integration test surfaced a classic Quarkus ARC gotcha — field access on `@ApplicationScoped` client proxies doesn't delegate to the underlying bean instance. The recording store's `signals` list was always empty when accessed via the field, even though method calls worked fine. Already in the garden as GE-20260505-84577e, but still catches people who haven't been bitten by it before.
