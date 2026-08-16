---
title: Teaching Agents to Notice Their Own Behaviour
date: 2026-08-05
author: Mark Proctor
tags: [casehub, engine, eidos, behavioural-contracts, routing, compliance]
---

# Teaching Agents to Notice Their Own Behaviour

CaseHub agents have always been routed — the platform decides which agent handles a task based on trust scores, capability matching, workload, and personality alignment. What they haven't done is notice when they're doing it badly.

The eidos framework recently shipped behavioural contracts: a way to declare what "doing it well" means for an agent capability, and a signal store that accumulates evidence of compliance or violation. The probe step checks the evidence and returns a health status. But the engine wasn't recording that evidence — the signal store was empty, and the probe had nothing to work with.

## Two dimensions that matter

We focused on two compliance dimensions that the platform can observe without any domain knowledge:

**Latency.** Every agent capability can declare a latency hint — "I expect to handle entity-resolution requests in about 1000ms." The engine already measures execution duration via Quartz job metadata. Comparing the two is straightforward: if the actual duration exceeds the hint multiplied by a violation threshold, record a `VIOLATED` signal. Otherwise, `COMPLIANT`. Both are recorded — the probe needs the ratio, not just the count.

**Attestation rate.** This one is simpler but has a longer tail. An agent that consistently declines or fails tasks is not meeting its attestation obligations for that capability. Success or completion counts as `COMPLIANT`. Declined, failed, or expired counts as `VIOLATED`. Over time, the signal store accumulates a compliance profile that the probe evaluates against configurable thresholds.

## The softest demotion

The routing question was interesting. Where does `BehavioralViolation` sit in the health hierarchy?

The eidos team designed it as the softest non-Ready status — softer than `EpistemicallyWeak`, much softer than `Degraded`. An agent with latency violations might still be the best available option for a capability. An agent with weak epistemic coverage is a bigger concern — it might not understand the domain at all.

The engine's `AgentHealth` enum now reflects this:

```java
public enum AgentHealth {
  READY,
  BEHAVIORAL_VIOLATION,
  EPISTEMICALLY_WEAK,
  DEGRADED
}
```

Routing strategies see the health level and can make nuanced decisions. The `AgentCandidate` record also carries the violations map — per-dimension counts that let a strategy treat latency violations differently from attestation-rate violations. A time-insensitive task might tolerate a slow agent; an unreliable one is a different problem.

## Closing the loop

The sealed `CapabilityStatus` hierarchy in eidos gained `BehavioralViolation` and `Excluded` alongside the existing `Ready`, `Degraded`, `Unavailable`, and `EpistemicallyWeak`. The engine's switch on this hierarchy used a `default` branch that silently mapped everything unknown to `READY` — including the new variants.

We replaced it with an exhaustive switch. No `default` branch. When eidos adds another variant, the engine won't compile until it handles it. That's the right failure mode — a compile error is always preferable to a silent wrong answer at runtime.

The recording itself follows the pattern established by `PersonalitySignalRecorder` and `GoalOutcomeRecorder` — a CDI bean injected into the completion handler, using `Instance<BehavioralSignalStore>` so deployments without eidos get transparent no-ops. The recorder runs on both success and failure paths, because both COMPLIANT and VIOLATED observations contribute to the compliance profile.

With this wired in, the feedback loop closes: agents execute tasks, the engine records compliance observations, the eidos probe evaluates the accumulated signals, and routing strategies incorporate the result. An agent that was fast last month but has been degrading gradually will start to see its `BehavioralViolation` health status influence where tasks are dispatched — without anyone writing a rule that says "avoid agent X." The system learns from evidence, which is what behavioural contracts were designed for.
