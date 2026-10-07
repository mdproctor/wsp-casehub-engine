## D1: Capability area architecture

**Choice:** Devtown areas alongside engine areas
**Alternatives:**
- Devtown areas only, disable engine defaults — cleaner health scores but requires suppression mechanism
- Devtown areas as specialisations of engine areas — reuses existing infrastructure but forces awkward conceptual mapping
**Rationale:** Full integration with the conductor's health scoring, regression detection, and compliance progression. Devtown's ImprovementConfig configures which areas are active for its case instances — engine defaults don't fire unless they have data sources.
**Trade-offs:** Must handle the case where engine default areas have no data in devtown context (they return inert/neutral assessments rather than dragging the score)
**Sources:** engine CapabilityAreaRegistry, EvolutionTicker health_refresh gate, #1148 design spec §1.1
**Exploration:** quick
**Status:** captured

## D2: Devtown capability areas — which 5 and what they measure

**Choice:** Core 5 — CI Reliability, Review Quality, Merge Queue Health, Reviewer Trust, SLA Compliance
**Alternatives:**
- Full 7 (add Code Churn + Contributor Health) — valuable but requires heavier data gathering (git history, cross-PR aggregation) not readily available
- Start with 3 (CI, Review, Queue) — tightest loop but misses trust and SLA dimensions
**Rationale:** These 5 map directly to devtown's existing data sources and cover the full PR lifecycle: code arrives (CI) → gets reviewed (Review Quality) → trust routes it (Reviewer Trust) → human gates fire (SLA) → it merges (Queue Health). Code Churn and Contributor Health are natural follow-ups.
**Trade-offs:** Omits code churn and contributor health, which are indirect signals. Can be added later without changing the architecture.
**Sources:** devtown ReviewFinding, MergeQueueService, TrustRoutingPolicy, SlaCalibrationService, CiRunnerWorker
**Exploration:** quick
**Depends on:** D1 (areas registered alongside engine defaults)
**Status:** captured

## D3: DevtownEvolutionApi facade — enrichment strategy

**Choice:** Enrichment at the view layer — DevtownEvolutionApi delegates to EngineEvolutionApi for all core operations, then enriches returned views with devtown context (PR links, CI dashboard URLs, reviewer names)
**Alternatives:**
- Enrichment via metadata — populate ImprovementRequest.metadata at proposal time, workbench renders generically. Stringly-typed, no semantic rendering.
- Enrichment at workbench level (frontend) — client resolves PR links via separate API calls. Two API calls per view, complexity in frontend.
**Rationale:** Clean separation. Engine API stays pure. Mutations pass through unchanged — only queries get enriched. Matches devtown's existing pattern of wrapping engine APIs with domain context.
**Trade-offs:** Requires thin wrapper record types (DevtownEvolutionStateSnapshot etc.) for the enriched views. Additional serialization layer.
**Sources:** devtown DefaultEngineCaseControlApi pattern, engine DefaultEngineEvolutionApi
**Exploration:** quick
**Depends on:** D1 (capability areas), D2 (which areas → what enrichment fields)
**Status:** captured
