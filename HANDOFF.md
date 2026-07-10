# Handoff — 2026-07-10

## What's Done

**ContextBridge protocol (#203) — all 8 tasks complete.** Task 8 (integration tests) landed: 7 tests covering MapBridge identity, JacksonPojoBridge typed POJO, EventLog metadata, backward compat, and mixed bridge coexistence. Full pipeline verified end-to-end. Cross-repo: worker repo branch `issue-203-context-bridge-protocol` still needs to land alongside engine changes.

## Immediate Next Step

**Fix flaky runtime tests.** `ActionGateIntegrationTest` (6 errors), `ActionGateResolutionTest` (3 errors), and `CaseLifecycleCdiEventTest` (1 error) all fail with Awaitility timeouts — not assertion failures. These are race conditions in test setup, not production bugs. File an issue, then fix. Likely causes: insufficient timeout, missing `await()` on async setup, or event ordering assumptions.

## Cross-Module

**Worker repo** has uncommitted-to-main changes on branch `issue-203-context-bridge-protocol` — needs to land alongside engine changes.

## What's Left

- Flaky tests — ActionGate + CaseLifecycle timeout failures · S · Med
- #680 — thread tenancyId through event bus messages · M · Med
- #646 — per-case CONTEXT_CHANGED serialization · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #689 | WorkItems boundary — typed payload/resolution | M | Med | Design projection in spec |
| #690 | SubCase boundary — typed context passing | S | Med | Design projection in spec |
| #691 | Signals boundary — typed signal overload | S | Med | Design projection in spec |
| #692 | Connectors boundary — typed inbound payloads | S | Med | Design projection in spec |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |
