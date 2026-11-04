---
title: "The Glue Between Governance and Mechanism"
date: 2026-07-29
author: Mark Proctor
entry_type: note
subtype: diary
status: draft
projects: [casehub-engine]
tags: [action-gate, multi-approver, oversight, spi-design, cross-repo]
---

# The Glue Between Governance and Mechanism

Three layers existed independently. The engine had `RiskDecision.GateRequired`
— a classifier tells the engine "this action needs human approval before the
case can proceed." The work repo had `WorkItemSpawnGroup` with full M-of-N
counting, OCC-protected counters, and assignment strategies. The UI had
`<blocks-approval-gate>` with quorum progress bars and per-voter status.

What was missing was about 200 lines of glue.

The interesting design question wasn't "how do we connect them" — it was
"where does the quorum decision live?" The answer falls out of first
principles: the domain classifier knows the risk. An AML classifier knows
SAR filings need dual-officer sign-off. A clinical classifier knows
treatment authorization needs 2-of-3 attending physicians. The quorum
requirement is part of the risk assessment, not the work infrastructure.

So `QuorumConfig` sits on `RiskDecision.GateRequired` in the engine-api tier.
The adapter translates governance semantics to work mechanism — mapping
`QuorumConfig(instances, required, onThresholdReached, allowSameAssignee)`
to `MultiInstanceConfig` at the boundary. Clean tiers, clean boundaries.

## The CallerRef Routing Trap

The one thing that would have been silently wrong: if children inherited
the parent's gate callerRef (`case:{caseId}/gate:{gateId}`), each child's
terminal event would trigger `ActionGateCompletionApplier`. The first child
to complete would approve the gate — defeating the entire point of M-of-N.

Children get no callerRef. Individual completions route through
`MultiInstanceCoordinator` for counter updates. The group terminal event
carries the parent's callerRef — that's what triggers gate resolution.
The design review caught and formalized this, but the constraint was
obvious once you traced the event routing.

## SpawnPort vs WorkItemCreator

The issue description mentioned SpawnPort. The spec said WorkItemCreator.
The design review challenged this — and the justification held:
`SpawnPort.spawn()` takes `ChildSpec(UUID templateId, ...)`, requiring
pre-registered templates. Gate approval items are programmatic — title,
payload, and scope vary per action. Creating a template just to immediately
override everything is ceremony, not design. `createMultiInstance()` on
`WorkItemCreator` is the programmatic counterpart to SpawnPort's
template-based spawning.

## The Chain Resolver Subtlety

`ChainedActionRiskClassifier.narrower()` picks the "most restrictive"
decision when multiple classifiers fire. Adding quorum comparison
required getting the ordering right: any quorum beats no quorum (obvious),
higher `required` beats lower (obvious), but equal `required`? Lower
`instances` is more restrictive — 2-of-2 unanimous is harder to achieve
than 2-of-5. The approval ratio matters, not the absolute count.

## What Followed

Once the core bridge landed, the follow-on issues fell quickly. Cancellation
cascade (parent obsoleted → all non-terminal children cancelled regardless
of status), approvedBy aggregation (query completed children's assigneeIds
at group completion), and typed resolution threading (store
`resolutionTypeName` on the parent WorkItem, resolve it from callerRef at
group completion). Each was a single-method addition to existing code paths.

The one I deferred was YAML schema for QuorumConfig. It looks simple —
parse a `quorum:` block in the case definition. But QuorumConfig is on
`RiskDecision.GateRequired`, which is a classifier output, not a definition
input. Adding it to the definition means introducing a "default quorum"
concept that the engine merges with classifier decisions. That's a design
question, not a parsing question.
