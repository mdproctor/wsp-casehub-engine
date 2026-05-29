# Handoff — 2026-05-29

**Head commit (engine):** cdbf6f0 — adr: 0003 AgentRoutingStrategy returns Uni<AgentAssignment>
**Head commit (workspace):** 00b860d — docs: add blog entry 2026-05-29-mdp01-routing-the-uncertain

## What Changed This Session

**engine#376 + engine#377 complete — PR#391 open on casehubio/engine.**

`AgentAssignment` record → sealed interface (`Assigned` / `Unresolvable` / `EscalateToOversight`).
`AgentRoutingStrategy.select()` → `Uni<AgentAssignment>` (reactive — blocking embedding calls safe from IO thread).
`TrustWeightedAgentStrategy` returns `EscalateToOversight` when all trust-eligible candidates are borderline (Phase 2, trust-maturity-model.md).
`TrustCandidateClassifier` — new `@ApplicationScoped` CDI bean shared by both trust strategies; `Phase` enum with `EXCLUDED_PHASE2B`/`EXCLUDED_PHASE3`; `OptionalDouble trustScore` (no NaN).
`AgentCandidateFactory` — shared static utility, eliminates duplication between two handlers.
`AgentRoutingEscalationHandler` — posts QUERY to oversight channel on escalation.
`casehub-engine-ai` — new optional module: `AgentEmbeddingProvider` SPI + `SemanticAgentRoutingStrategy` @Priority(2).

**Four issues filed this session:**
- engine#383 — oversight response loop (COMMAND → re-trigger routing)
- engine#384 — PlanItem state during escalation
- engine#385 — embedding vector cache
- engine#386 — LangChain4j AgentEmbeddingProvider implementation

ADR-0003 recorded. PP-20260529-9f9627 (spi-reactive-blocking-io) captured in garden.
Blog: `blog/2026-05-29-mdp01-routing-the-uncertain.md`

## Immediate Next Step

Wait for PR#391 review and merge. Next work item: **engine#274** — BlackboardRegistry hydration from PlanItemStore on restart.

## What's Left

- engine#274 — BlackboardRegistry hydration from PlanItemStore on restart · M · Med
- engine#383 — Oversight response loop: COMMAND → re-trigger routing · M · Med
- engine#384 — PlanItem state during escalation (ESCALATING state?) · M · Med
- engine#385 — Embedding vector cache · S · Low
- engine#386 — LangChain4j AgentEmbeddingProvider implementation · S · Low
- parent#87 — PLATFORM.md capability table stale (WorkBroker references) · S · Low
- parent#88 — PLATFORM.md casehub-engine-ai and AgentEmbeddingProvider · S · Low
- engine#388 — Minor code quality items (classifier decide() instance, eidos-api dep, etc.) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| engine#383 | Oversight response loop: COMMAND from human re-triggers routing | M | Med | Depends on PR#391 merge |
| engine#385 | Embedding cache for SemanticAgentRoutingStrategy | S | Low | After PR#391 merge |
| engine#386 | LangChain4j AgentEmbeddingProvider implementation | S | Low | After PR#391 merge |

## Key References

- PR: casehubio/engine#391
- Blog: `blog/2026-05-29-mdp01-routing-the-uncertain.md`
- Spec: `proj/docs/specs/2026-05-28-semantic-routing-escalation-design.md`
- ADR: `proj/docs/adr/0003-agent-routing-strategy-reactive-spi.md`
- Protocol: PP-20260529-9f9627 — garden casehub/spi-reactive-blocking-io.md
