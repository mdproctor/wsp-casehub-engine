# Handoff — 2026-06-14

**Head commit (engine main):** 7e28f676 — feat: casehub-engine-inbound — MessageReceivedEvent → WorkItem bridge (engine#468 + engine#469)
**PR:** #492 open (casehub-engine-inbound bridge module — awaiting merge)
**CI:** not yet run on PR

## What Changed This Session

Built `casehub-engine-inbound` — a new optional module bridging qhorus `MessageReceivedEvent` to casehub-work WorkItems. `InboundWorkItemBridge implements MessageObserver`; consumer apps provide `InboundWorkItemPolicy` to decide per-message whether and how to create a WorkItem. Bridge is inert without a policy. 10 tests green. Spec through two rounds of review before implementation. Closed #468 and #469.

Two bugs found in spec before implementation:
- `ifPresent()` inside try/catch absorbs lambda exceptions (infrastructure failures logged as policy failures) — required separating the policy call
- Excluding `JpaWorkloadProvider` without `StubWorkloadProvider` fails CDI boot — `WorkItemAssignmentService` direct-injects `WorkloadProvider` (not `Instance<>`)

## What's Left

- parent#243 — add casehub-engine-inbound to engine deep-dive module table · S · Low
- parent#244 — sync PLATFORM.md cross-repo dependency rows · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #492 | Merge PR — casehub-engine-inbound bridge | XS | Low | Awaiting CI + review |
| #486 | Thread tenancyId through WorkerRetriesExhaustedEvent and ActionGateWorkerFaultedEvent | S | Low | Filed; unblocked |
| #491 | Fix @QuarkusTest investigations stuck in RUNNING | S | High | P0 — all AML tests affected; three suspects |
| #476 | ImplementationRoutingStrategy SPI | M | Med | Blocks ledger#136 |

## Key References

- Blog: `blog/2026-06-14-mdp01-the-bridge-and-the-try-catch-that-lied.md`
- Spec: `docs/specs/2026-06-13-inbound-workitem-bridge-design.md` (in project repo)
- Garden: GE-20260427-5d7c67 revised (WorkloadProvider injection chain + test inner class activation)
- Garden: GE-20260614-21317a (new — ifPresent inside try/catch silently reclassifies exceptions)
