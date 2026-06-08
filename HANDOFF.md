# Handoff — 2026-06-08

*Updated: devtown#62 closed — removed from backlog. parent#188 closed — removed from backlog.*

**Head commit (engine):** 508b8dc — fix: adapt to TrustGateService and AgentDescriptor API changes — unblock CI
**Head commit (workspace):** b926672 — fix: promote corrected mdp01 frontmatter to main
**Both repos on:** main
**PR merged:** casehubio/engine#443 — dynamic candidateGroups/Users for humanTask (engine#387)

## What Changed This Session

**engine#387 — dynamic candidateGroups/candidateUsers for humanTask: shipped and merged.**

`candidateGroups` and `candidateUsers` in humanTask YAML bindings now accept JQ expressions evaluated against the case context at event-publish time. `ListEvaluator` sealed interface (`StaticList`/`JQList`) keeps the type hierarchy clean — separate from `ExpressionEvaluator` (which is a boolean predicate). `ListExpressionResolver @ApplicationScoped` handles JQ evaluation; resolution failure blocks the event. ADR-0008 records the hierarchy decision. PR #443 green and merged after fixing two pre-existing CI failures: `TrustGateService.findScore()` → `currentScore()` in actor-state, and `AgentDescriptor` constructor arity in engine-ai tests.

**engine#442 filed — universal routing architecture.** Post-merge discussion surfaced that `ListEvaluator` is a sealed dead-end for richer routing strategies (Drools, ML). Opened as a platform-coherence design initiative: audit all routing decision points, design a named-strategy SPI pattern, document in PLATFORM.md and protocols. Affects engine#439 scope.

## Immediate Next Step

engine#383 — Oversight response loop. M · Med · Unblocked. Run `/work engine#383`.

## Cross-Module

**Unblocked by engine#387 merge** (these can now proceed):
- `aml` — aml#42: SAR filing, account freeze, law enforcement referral · L · Med
- `clinical` — clinical#47: SUSAR filing, dose modification, patient withdrawal · L · Med
- `devtown` — devtown#56: production deploy, contributor access, security escalation · M · Med
- `life` — life#20: spend threshold, non-refundable bookings, contractor instruction · M · Low
- `openclaw` — openclaw#6: oversight channel gate (Epic 6 end-to-end wiring) · L · High

## What's Left

- engine#433: persist `pendingActionGate` in `CaseInstanceEntity` (restart resilience) · M · Med
- engine#434: integration test for classifier-throws fail-safe · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#383 | Oversight response loop | M | Med | Immediate |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#442 | Universal routing architecture design | L | High | Design-first; affects engine#439 |
| engine#404 | Registry lifecycle design | L | High | Design-only |
| — | AI Fusion typed fact space | XL | High | New module — own session |

## Key References

- PR: https://github.com/casehubio/engine/pull/443 (merged)
- Spec: `docs/specs/2026-06-07-humantask-dynamic-candidate-groups-design.md`
- ADR: `docs/adr/0008-list-evaluator-separate-sealed-hierarchy.md`
- Blog: `blog/2026-06-07-mdp02-humantask-dynamic-routing.md`
- engine#442: https://github.com/casehubio/engine/issues/442 (universal routing architecture)
