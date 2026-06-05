---
layout: post
title: "The Mixed-Pool Gap"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [trust-routing, bootstrap, escalation, mutiny]
---

The initial spec for the bootstrap guard had a gap I missed.

The feature itself is clear enough: certain capabilities — `merge-executor`, `security-review` — are irreversible enough that assigning an unproven agent is genuinely dangerous. `TrustWeightedAgentStrategy` will assign a BOOTSTRAP-phase agent if it has the best workload score, because BOOTSTRAP candidates score by availability and there's nothing to stop them. Adding `bootstrapEscalationRequired` to `TrustRoutingPolicy` is the fix.

What I got wrong was the guard condition. My spec proposed `allMatch(Phase.BOOTSTRAP)`: if every candidate is BOOTSTRAP, escalate. The spec reviewer caught the gap — a pool of [BOOTSTRAP (workload 1.0), BORDERLINE (score 0.0)] doesn't fire the guard because BORDERLINE is present. Then `decide()` assigns the BOOTSTRAP agent because 1.0 beats 0.0. The borderline-escalation path also fails to fire: it requires all candidates to score 0.0, and BOOTSTRAP's positive workload score prevents that. Two safety nets, both bypassed in the same pool.

The right condition is `!hasQualified && hasBootstrap`. If no QUALIFIED agent exists and at least one BOOTSTRAP agent does, escalate. This handles the all-BOOTSTRAP case, the BOOTSTRAP+BORDERLINE case, and the BOOTSTRAP+EXCLUDED case. When QUALIFIED agents exist alongside BOOTSTRAP ones, we strip the BOOTSTRAP candidates from the scoring pool — a busy QUALIFIED agent beats an idle BOOTSTRAP agent, which is the whole point of the flag.

Claude and I hit a constraint in `SemanticAgentRoutingStrategy` that wasn't obvious from the spec. That strategy dispatches scoring to `Infrastructure.getDefaultWorkerPool()` via `emitOn()` because embedding computation is CPU-bound. If we placed the bootstrap guard inside the lambda, the strategy would enter the worker pool before discovering it should have escalated — paying the embedding cost for a result that gets thrown away. The fix: compute the eligible list and run the pre-screen before `emitOn()`. The lambda captures the pre-filtered list; BOOTSTRAP candidates never reach the embedding step when the flag is set.

The spec review caught twelve issues before implementation. Most were naming and documentation — `ALL_BORDERLINE` was wrong (the condition is "any borderline in an all-zero pool", not "all candidates borderline"), `NO_QUALIFIED_AGENT` Javadoc was imprecise about when scoring occurs, `AgentAssignment.EscalateToOversight`'s Javadoc had stale wording. One was genuinely operational: the `[METRIC:escalation.no-qualified-agent]` log was inside `postQuery()`, which only runs when an oversight channel is found. The critical scenario — bootstrap guard fired, no channel — would have produced no metric. The log had to move before the channel lookup.

We ended up with `BORDERLINE_STALEMATE` and `NO_QUALIFIED_AGENT` as the two escalation reasons, both carried through `EscalateToOversight` and `AgentRoutingEscalationEvent`. The handler produces distinct QUERY text per reason. A human reading the oversight channel now knows whether the pool has borderline agents or simply unproven ones — not the same problem, and not the same intervention.

The devtown follow-up is tracked in devtown#62: `DevtownTrustRoutingPolicyProvider` needs `bootstrapEscalationRequired = true` for `merge-executor`, `security-review`, and `architecture-review`.
