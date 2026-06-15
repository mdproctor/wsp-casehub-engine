# Handoff — 2026-06-15

**Head commit (engine main):** 27875650 — fix(engine): JQ eval against working panel + fire-and-forget CDI + trigger context + CaseOutcomeObserver + ActionGatePolicy
**PR:** #495 open (upstream)
**CI:** engine 740 tests green; AML verified manually

## What Changed This Session

Root-caused #491 — panels refactor (`734ed65e`, June 9) changed `asJsonNode()` from flat working data to panel document. All JQ evaluation in the engine evaluated against the full document, silently breaking every consumer YAML binding. Fixed 8 production files + 55 test files. Also landed: fire-and-forget CDI events (#491 secondary), trigger context threading (#231), CaseOutcomeObserver SPI (#477), ActionGatePolicy enum (#472). PR #495 open against upstream.

Cross-repo: filed casehubio/aml#63 — AML needs `QhorusInboundCurrentPrincipal` excluded from test CDI and drain mechanism updated.

## What's Left

- PR #495 — awaiting CI/merge on upstream · XS · Low
- parent#243 — add casehub-engine-inbound to engine deep-dive module table · S · Low
- parent#244 — sync PLATFORM.md cross-repo dependency rows · S · Low
- aml#63 — exclude QhorusInboundCurrentPrincipal + update drain mechanism · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #486 | Thread tenancyId through WorkerRetriesExhaustedEvent and ActionGateWorkerFaultedEvent | S | Low | Filed; unblocked |
| #476 | ImplementationRoutingStrategy SPI | M | Med | Blocks ledger#136 |

## Key References

- Blog: `blog/2026-06-15-mdp01-the-wrong-hypothesis.md`
- Garden: GE-20260615-35f52f (panels JQ silent break — 14/15)
- Garden: GE-20260615-c234fc (@DefaultBean without quarkus-arc — 10/15)
- Garden: GE-20260615-c5340d (reproduce in consumer app technique — 9/15)
