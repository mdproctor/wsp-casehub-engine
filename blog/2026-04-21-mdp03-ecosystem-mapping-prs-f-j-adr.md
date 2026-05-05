---
layout: post
title: "Ecosystem Mapping, PRs F-J, ADR-0002"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [blackboard, architecture, taskbroker, workitems]
---

The session started with four parity gaps and ended with fifteen GitHub issues
mapped against the full ecosystem. That's a wider arc than planned.

The gap closure was straightforward. PRs F through J: `BlackboardPlanConfigurer`
(the pre-registration mechanism treblereel had; we didn't), strict `PlanItem`
lifecycle (`markRunning/markCompleted` throwing on invalid transitions — the
strict API immediately caught test code jumping PENDING→COMPLETED without going
through RUNNING), `SubCase` data model, `Stage.builder()` requiring an explicit
entry condition, and the binding gating pattern from the Kogito flexible process
model. That last one became ADR-0002. The decision — presence of
`stage.addBinding()` IS the opt-in, no flag needed — was the right one, and it
held up well under questioning about whether all four approaches could equally
work. They can't. Approach C is the only one that's both bounded and fair; A is
unbounded; B punishes late claimants; D is a pragmatic middle ground.

The SLA continuation question got its own issue (#122) because it deserved its
own lifecycle, not a comment on a design document. I put it in the wrong place
the first time.

The ecosystem mapping came from a single question about the TaskBroker and
WorkItems integration. That thread kept pulling: how the two systems compose,
where the deadline enforcement boundary sits (TimeoutEnforcer fires; WorkItems
owns the escalation policy), the claim model atomics, the `HumanEscalationPolicy`
interface. Fifteen issues total — one for each use case where casehub + Qhorus
+ quarkus-ledger + Claudony can do something no current Java framework does. The
canonical comparison against LangChain4j anchors each one without framing it
as competitive. These are future work trackers, not commitments.

PropagationContext and the architectural exclusion tests went cleanly. The fitness
tests asserting that `NotificationService` and `CaseFileContribution` are never
ported — two test methods, five forbidden class names — are the right level of
investment for confirming a decision.
