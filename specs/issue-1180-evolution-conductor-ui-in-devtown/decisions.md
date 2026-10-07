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

## D4: Evolution tab integration in devtown dashboard

**Choice:** 9th tab with hostPanel — registerPanel + hostPanel following existing pattern, endpoint at /api/devtown/evolution, positioned after System before Definitions
**Alternatives:**
- Nested under System tab — fewer top-level tabs but undersells a major feature surface
- Standalone page (separate route) — more screen real estate but breaks devtown's single-dashboard pattern
**Rationale:** Follows devtown's existing dashboard pattern. The workbench is substantial enough for a top-level tab. Wiring is trivial — registerPanel + configure() with endpoint.
**Trade-offs:** 9 tabs is getting crowded. If the tab count grows further, devtown may need tab grouping or a navigation redesign — but that's not this issue's concern.
**Sources:** devtown index.ts registerPanel/hostPanel pattern, blocks-evolution-workbench configure() API
**Exploration:** quick
**Depends on:** D3 (DevtownEvolutionApi at /api/devtown/evolution)
**Status:** captured

## D5: Devtown ImprovementCategoryProvider — review pipeline categories

**Choice:** Review pipeline improvement categories — 5 devtown-specific categories (reviewer-calibration, routing-adjustment, sla-tuning, gate-tightening, queue-optimization) with a shorter 5-stage pipeline (analyze → propose → review → apply → observe)
**Alternatives:**
- Mirror code-evolution categories for devtown's codebase — reuses existing definitions but doesn't leverage devtown's unique domain data. Any CaseHub app could do this.
**Rationale:** Makes devtown the genuine "first consumer" — the conductor improves devtown's review orchestration, not just its source code. Each category has a clear actionable improvement the conductor can drive.
**Trade-offs:** Some actions (trust weight adjustment, routing changes) may be too impactful for full automation initially — they'd start with GATED gate policy. Shorter stage pipeline means less granular gating but faster improvement cycles.
**Sources:** engine ImprovementCategoryProvider SPI, CodeEvolutionCategoryProvider pattern, #1148 design spec §1.1
**Exploration:** quick
**Depends on:** D1 (areas alongside engine defaults), D2 (5 areas drive category relevance)
**Status:** captured

## D6: Deep links — how the workbench renders devtown context

**Choice:** Custom workbench tabs via TabDefinition extension — existing tabs render enriched strings (target becomes "PR #42: Fix auth middleware"), devtown adds 1-2 custom tabs for domain-specific views. No blocks-ui changes needed.
**Alternatives:**
- Blocks-ui component extension via render callbacks — more flexible but requires blocks-ui changes and adds API surface
- Devtown-specific component wrappers — full rendering control but duplicates blocks-ui logic and diverges over time
**Rationale:** The workbench's existing tabs already render target, category, stage as strings. Enriching those strings at the API layer makes them meaningful without touching blocks-ui. Custom tabs via TabDefinition[] handle anything the base tabs can't show.
**Trade-offs:** PR links won't be clickable in the Streams tab (they're rendered as text, not anchor tags). Acceptable for v1 — clickable links can be added later via blocks-ui render callbacks if needed.
**Sources:** blocks-evolution-workbench TabDefinition[] extension, EvolutionWorkbenchProps.tabs
**Exploration:** quick
**Depends on:** D3 (view-layer enrichment), D4 (tab integration)
**Status:** captured
