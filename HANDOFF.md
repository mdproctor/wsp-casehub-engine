# Handoff — 2026-07-10

## What's Done

**ContextBridge protocol (#203) — Tasks 1–7 of 8 implemented.** Full pipeline wired: `WorkerFunction<T>` parameterised with Reified Varargs Type Token DSL, `ContextBridge<T>` SPI with 3 built-in bridges, `BridgeResolver` (CDI discovery chain), pipeline signatures widened to `Object`, `WorkerScheduleEventHandler` and `QuartzWorkerExecutionJob` integrated with bridge initialise/serialise/deserialise/extractOutput. YAML `contextType` support added. Cross-repo: worker (1 commit), engine (7 commits), workers (1 commit on main).

**Key architecture decision this session:** `BridgeResolver` moved from `runtime` to `common/internal/context/` — `scheduler-quartz` can't depend on `runtime` (wrong direction). Shared infrastructure that both modules consume goes in `common`.

## Immediate Next Step

Resume implementation on branch `issue-203-context-bridge-protocol`. **Task 8 only: integration tests.** Read the plan at `docs/plans/2026-07-09-context-bridge-implementation.md` (Task 8 section) and the spec at `docs/specs/2026-07-09-context-bridge-architecture.md` (§Combinatorial Test Matrix). Create `runtime/src/test/java/io/casehub/engine/ContextBridgeIntegrationTest.java` as a `@QuarkusTest`. Test Pattern 1 (MapBridge identity), Pattern 2 (typed POJO via JacksonPojoBridge), EventLog metadata `contextBridgeType`, and backward compat (pre-bridge EventLog entries deserialise to Map). **Use IntelliJ MCP for all code operations.**

## Cross-Module

**Worker repo** has uncommitted-to-main changes on branch `issue-203-context-bridge-protocol` — needs to land alongside engine changes. Workers repo change is already on `main`.

## What's Left

- #203 Task 8 — integration tests for ContextBridge · M · Med
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
