# Handoff — 2026-07-02

## What's Done

**Closed 2 issues:**
- **#609**: AgentCandidateFactory subsumption matching via VocabularyRegistry — CDI conversion, two-tier matching, NoOpVocabularyRegistry @DefaultBean
- **#528** (closed as already resolved): WorkerFunction.Flow extraction — confirmed fully resolved by #567

**Cross-repo commits (blocks consolidation):**
- `3cdb1f90` migrated oversight SPIs from engine-api to casehub-blocks
- `fc6938fa` restored oversight SPIs + added routing package to engine-api (blocks#23, #17)

**#634 Phase 1 complete:** universal routing strategy audit — systematic audit of all ~64 routing mechanisms across engine, work, qhorus, eidos, connectors. Design recommendation: NamedStrategy marker + StrategyResolver + domain-specific SPIs (not a universal contract).

## Cross-Module

**Consumer repos still need capabilityNames migration:**
- casehub-aml#85, casehub-devtown#117

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #634 | Universal pluggable routing — Phase 2 design spec | L | High | Phase 1 audit done; design direction agreed |
| #629 | WorkerRecoveryHealthCheck @Liveness → @Readiness | XS | Low | Flagged by trebreel |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap |

## References

- Routing audit: `docs/specs/2026-07-02-universal-routing-strategy-audit.md`
- Subsumption design: `docs/specs/2026-07-02-agent-candidate-subsumption-design.md`
