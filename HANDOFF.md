# Handoff — 2026-06-16

**Branch:** `issue-493-signal-stage-routing`
**PR:** #499 open (upstream)

## What Changed This Session

Implemented four engine changes from the signal-stage-routing design spec: signal API returns `CompletionStage<Void>` (#493), `ImplementationRoutingStrategy` SPI for capability binding selection (#476), repeatable Stage lifecycle (#482), and Stage-PlanItem auto-registration (#497). CDI lifecycle events standardised to fire-and-forget across all handlers. 25 files changed, 1014 insertions. All tests green (263 blackboard + 20 signal + 7 routing/selection).

Filed engine#497 (auto-registration precursor) and engine#498 (protocol update for CDI fire-and-forget pattern).

## What's Left

- PR #499 — awaiting review/merge on upstream · M · Med
- engine#498 — update engine-cdi-event-await-chain protocol · XS · Low
- parent#243 — add casehub-engine-inbound to engine deep-dive module table · S · Low
- parent#244 — sync PLATFORM.md cross-repo dependency rows · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #486 | Thread tenancyId through WorkerRetriesExhaustedEvent and ActionGateWorkerFaultedEvent | S | Low | Filed; unblocked |
| — | TrustWeightedImplementationStrategy in casehub-engine-ledger | M | Med | Symmetric with TrustWeightedAgentStrategy |
| — | QuarkMind migration — delete StrategyTrustRouter (~300 lines) | S | Low | Depends on #476 merge |

## Key References

- Design spec: `docs/specs/2026-06-15-signal-stage-routing-design.md`
- Blog: `blog/2026-06-15-mdp01-the-wrong-hypothesis.md` (previous session)
