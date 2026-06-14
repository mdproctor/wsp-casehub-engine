# Handoff — 2026-06-14

**Head commit (engine main):** 9dcb8d24 — Merge pull request #492 from mdproctor/main
**PR:** #492 merged ✅
**CI:** green

## What Changed This Session

Fixed CI on PR #492 — new `@Default CurrentPrincipal` bean (`QhorusInboundCurrentPrincipal`) appeared in qhorus SNAPSHOT 2026-06-14, creating CDI ambiguity with `TenantScopedPrincipal` (casehub-work) in `actor-state` test classpath. Fix: added `QhorusInboundCurrentPrincipal` to `quarkus.arc.exclude-types` in `actor-state/src/test/resources/application.properties`. PR #492 merged; upstream main synced.

Also updated casehub-work HANDOFF — removed engine#468 and engine#469 (done).

## What's Left

- parent#243 — add casehub-engine-inbound to engine deep-dive module table · S · Low
- parent#244 — sync PLATFORM.md cross-repo dependency rows · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #491 | Fix @QuarkusTest investigations stuck in RUNNING | S | High | P0 — all AML tests affected; three suspects |
| #486 | Thread tenancyId through WorkerRetriesExhaustedEvent and ActionGateWorkerFaultedEvent | S | Low | Filed; unblocked |
| #476 | ImplementationRoutingStrategy SPI | M | Med | Blocks ledger#136 |

## Key References

- Blog: `blog/2026-06-14-mdp01-the-bridge-and-the-try-catch-that-lied.md`
- Spec: `docs/specs/2026-06-13-inbound-workitem-bridge-design.md` (in project repo)
- Garden: GE-20260427-5d7c67 revised (WorkloadProvider injection chain + test inner class activation)
- Garden: GE-20260614-21317a (ifPresent inside try/catch silently reclassifies exceptions)
