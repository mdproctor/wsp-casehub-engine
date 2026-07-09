*Updated: #681 closed — removed from backlog.*

# Handoff — 2026-07-09

## What's Done

**ContextBridge protocol (#203) designed, spec'd, and adversarially reviewed.** Architecture doc at `docs/specs/2026-07-09-context-bridge-architecture.md` — 5 rounds, 18 issues, all verified ($23.20). Implementation plan at `docs/plans/2026-07-09-context-bridge-implementation.md` — 8 tasks. Branch `issue-203-context-bridge-protocol` created, codebase clean and ready for implementation.

**Key decisions:** ContextBridge<T> SPI in engine-api (not platform-api — adversarial debate confirmed). Reified Varargs Type Token for `fn().apply()` DSL. PropagationContext removed from bridge signature (engine-level threading). `initialise()` takes `JsonNode narrowedInput` (engine evaluates JQ, bridge receives result). Four tracking issues created for deferred boundaries (#689-#692).

## Immediate Next Step

Resume implementation on branch `issue-203-context-bridge-protocol`. Run `/work` to continue. **Critical: use IntelliJ MCP for ALL code operations** — see memory entry `feedback_intellij_mcp_mandatory.md`. Task 1 (WorkerFunction<T> parameterisation) was attempted with bash/Edit and reset — must restart with IntelliJ-first.

## What's Left

- #203 implementation — 8 tasks in plan, 0 completed · L · Med
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
