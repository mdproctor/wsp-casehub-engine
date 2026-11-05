# HANDOFF — casehub-engine

## Last Session (2026-11-05, session 9)

### agency#383 — Per-agent degradation + cascade policy (complete)

Designed and implemented a two-level degradation model for the coordination topology system. Per-loop: composable `DegradationTrigger` SPI checks budget exhaustion and latency breaches before each LLM call, switching to heuristic fallback via `DegradationAwareSummariser` decorator. Cascade: when enough loops degrade, the topology responds with a one-way structural change — CentralisedTopology forces all-heuristic, GraduatedTopology forces STIMULUS_DRIVEN scope, HiveTopology simplifies orchestration pattern.

Key design decision: degradation is stateless per-call (trigger IS the state), cascade is a one-way ratchet. Push model via `SwarmDegradedEvent` to new `CoordinationTopology.onDegradation()` — pull model had a dead path where budget-exhausted loops stop producing feedback.

Branch: `issue-383-per-agent-degradation-cascade-policy` in agency repo (12 commits). Engine: `TopologyContext` gained `DegradationPolicy` field. Ready for `work end`.

## Immediate Next Step

Run `work end` on agency#383 to close the branch, then advance to #384 (DecisionLoopEvent observability).

## Cross-Module

- **agency#391/#392** — topology test assertions still blocked on pre-existing compilation errors in agency-core topology classes

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-383-per-agent-degradation-cascade-policy/2026-11-04-per-agent-degradation-cascade-policy-design.md` |
| Decisions | `specs/issue-383-per-agent-degradation-cascade-policy/decisions.md` |
| Implementation plan | `plans/2026-11-04-per-agent-degradation-cascade-policy.md` |
| Plan | `/Users/mdproctor/claude/casehub/slots/220/.plan` |
