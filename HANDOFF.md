# Handoff — 2026-05-28

**Head commit (engine):** 618e233 — feat: promote design spec from issue-371-337-336-spi-trust
**Head commit (workspace):** 83de390 — docs: mark closed, deletion due 2026-06-11

## What Changed This Session

**engine#371, #337, #336 complete — PR#378 merged.**
`CaseLifecycleEvent` promoted to `io.casehub.engine.common.spi.event` (#371).
Engine-owned `AgentRoutingStrategy` SPI replaced borrowed `WorkerSelectionStrategy`/`WorkBroker` from casehub-work; `casehub-work-api` and `casehub-work-core` removed from runtime/pom (#337).
`TrustScoreCache` + `TrustWeightedAgentStrategy` (four-phase trust maturity model, CAPABILITY + CAPABILITY_DIMENSION scores) in `casehub-engine-ledger` (#336).
Rebase conflict resolved with Dmitrii's `engine.common.*` package reorganization (PR#375).

**Four issues filed this session:**
- `casehub-work#231` — ClaimFirstStrategy → @Alternative @Priority(0)
- `engine#376` — SemanticAgentRoutingStrategy
- `engine#377` — borderline agent escalation
- `parent#80` — engine deep-dive doc sync

**PR#366 (engine#349 signal bridge) confirmed merged.**

## Immediate Next Step

`engine#274` — BlackboardRegistry hydration from PlanItemStore on restart. Next actionable piece of engine work.

## What's Left

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med
- casehub-work#231 — ClaimFirstStrategy → @Alternative @Priority(0) (filed this session) · XS · Low
- parent#80 — engine deep-dive doc sync (casehub-engine.md) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| engine#376 | SemanticAgentRoutingStrategy — embedding-based agent selection | M | Med | Unblocked by #337 |
| engine#377 | Borderline agent escalation — human oversight path | S | Med | Unblocked by #336 |
| casehub-work#231 | ClaimFirstStrategy → @Alternative @Priority(0) | XS | Low | Separate session |
| parent#80 | engine deep-dive doc sync (casehub-engine.md) | S | Low | Separate session |

## Key References

- Spec: `specs/issue-371-337-336-spi-trust/2026-05-27-agent-routing-strategy-design.md`
- Blog: `blog/2026-05-28-mdp01-the-wrong-abstraction.md`
- Garden: REVISE GE-20260423-daef97 — resolveObserverMethods() double-silence amplifier
- Branch closed: `issue-371-337-336-spi-trust` (deletion due 2026-06-11)
