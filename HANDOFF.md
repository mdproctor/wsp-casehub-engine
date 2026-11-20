# HANDOFF — casehub-engine

## Last Session

Implemented #1331 (auto-discovery YAML registration), brainstormed #1332 (CollectionStore platform SPI — created slot 226 for cross-repo work), scaffolded #1333 (agent worker function provider).

### What Landed

Branch `issue-1331-auto-discovery-classpath-yaml` landed on main (2 commits):

1. **feat(#1331):** Enhanced `CaseYamlRegistrar` with full pipeline loading — injected `ExpressionEngineRegistry`, `WorkerFunctionProviderRegistry`, `@YamlMapper ObjectMapper`, and `AgentProvider`. Uses `CaseDefinitionYamlMapper.load(JsonNode, ...)` for full worker/expression wiring. Added overlay composition via convention path (`-overrides` suffix). Made `YamlCaseHub.deriveConventionPath()` public.
2. **test(#1331):** Overlay composition test verifying convention-path merging. Promoted specs and blog to `docs/`.

### Design Outcome — #1332 (CollectionStore)

The brainstorming surfaced a platform-level design: a `CollectionStore` SPI (typed document CRUD + concurrency over named collections). Not JSON-specific — JSON is a serialization detail of one backend. Cases and app data (customers, orders, lookups) share the same data layer, enabling natural joins. Full cross-repo implementation in slot 226 (`/Users/mdproctor/claude/casehub/slots/226` — repos: engine, parent, pages).

### Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 1336 | Integration test: verify auto-discovered YAML wires do: block workers via FlowWorkerFunctionProvider | XS | Low |

### Blog

"The YAML That Didn't Need a Class" — at `docs/blog/2026-11-20-mdp01-the-yaml-that-didnt-need-a-class.md`.

## Active Work

Branch `issue-1333-agent-worker-function-provider` scaffolded. Next session: brainstorm and implement `call: agent` worker function provider for eidos dispatch + `agent-task:` binding resolver. Issue spans engine and eidos.

## Pre-Existing State

Engine has stashed changes from prior sessions (multiple stash entries). These predate all current work and are NOT part of any active branch.

## References

- `docs/specs/issue-1331-auto-discovery-classpath-yaml/2026-11-19-case-auto-discovery-design.md`
- `docs/blog/2026-11-20-mdp01-the-yaml-that-didnt-need-a-class.md`
- Slot 226: `/Users/mdproctor/claude/casehub/slots/226` (#1332 CollectionStore)
