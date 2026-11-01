# HANDOFF — casehub-engine

## Last Session (2026-11-01)

### Epic #1274 — Hive Mind integration wiring — COMPLETE

Implemented all 6 child issues, squashed to 6 clean commits, landed on main:

1. `6ed53ad64` — #1277: SwarmCoordinationListener exception propagation
2. `81432671b` — #1278: onMemberCompleted wired into departure flow
3. `9a7d3df04` — #1276: volatile fields for swarm state visibility
4. `2b94e181d` — #1275: SignalRegistry thread safety (compute + purge)
5. `d6f197fab` — #1279: eidos proximity queries (3 integration points)
6. `0ebf4ce7f` — #1280: qhorus broadcast (9 event types, CDI + Spring producers)

Branch audit caught a real gap: StigmergyCoordinator had no CDI/Spring bean producer — pattern detection events would silently not post in production. Fixed by adding `@Produces`/`@Bean` methods in RuntimeBeans and RuntimeManualConfig.

### Issue triage

Closed stale issues this session:
- #1237, #1243, #1236 — langchain4j removal (already done via #1245/#1252)
- #1027 — DSL audit epic (all 9 children already closed)
- #1099, platform#324, ras#65 — CBR ensemble / shared algorithms (superseded by simulated responses, eidos routing, neocortex BJT)

### Next work — branch `issue-1268-runtime-enhancements`

.plan queued with 3 issues:
1. **#1268** (S/Low) — FlowWorkerFunctionHandler PartialSuccess for mixed step results ← active
2. **#1234** (S/Med) — org-aware escalation in AgentRoutingStrategy
3. **#1269** (M/Med) — API projections for gallery (AgentDescriptor + CoordinationPipeline)

## Immediate Next Step

`work continue` → brainstorm + implement #1268.

## Cross-Module

- GITHUB_TOKEN env var is invalid — blocks `gh` and `git push` unless prefixed with `env -u GITHUB_TOKEN`. Keyring auth (mdproctor) works when GITHUB_TOKEN is unset.
