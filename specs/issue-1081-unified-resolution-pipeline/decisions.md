# Decisions — Unified Resolution Pipeline (#1081)

## D1: Overall architecture

**Choice:** Distributed composition — no new module
**Alternatives:**
- New `casehub-engine-resolution` module — creates a module boundary where none is needed; the pipeline is a conceptual flow, not a deployable unit
- Neocortex-level unification — violates boundary between platform infrastructure and domain coordination
**Rationale:** The pipeline is already distributed across CbrRetrievalService (retrieval), WorkflowExecutionCompletedHandler (feedback), JudgmentTarget (presentation). Each piece lands in its natural home. The engine's handler-driven, event-bus-mediated architecture already composes these concerns.
**Trade-offs:** No single class representing "the resolution pipeline" — understanding the flow requires tracing across handlers.
**Sources:** engine CLAUDE.md (CBR Retrieval Bridge section), neocortex CLAUDE.md (module structure)
**Exploration:** quick
**Status:** captured

## D2: Corpus source model

**Choice:** SPI-driven — CorpusSourceAdapter in engine, consumers implement for their source
**Alternatives:**
- Filesystem default — ships a concrete adapter; too opinionated for a platform
- API-based ingestion — REST endpoint only; misses change detection and batch ingestion
**Rationale:** The engine is a platform library, not an application. It should define the SPI and let consumers wire their source (filesystem, S3, API, database).
**Trade-offs:** No out-of-box ingestion — consumers must implement the adapter.
**Sources:** neocortex corpus-api (CorpusStore, ChangeSource SPIs)
**Exploration:** quick
**Status:** captured

## D3: Human-in-the-loop model

**Choice:** Enrich JudgmentTarget with resolution candidate data
**Alternatives:**
- New ResolutionTarget binding type — duplicates JudgmentTarget's scheduling/completion infrastructure
- Routing strategy enrichment — blurs the line between routing and judgment
**Rationale:** JudgmentTarget already handles human selection with structured outcomes, verification, escalation, and CallerConfig (Human/Llm/A2A). Resolution candidates are a richer payload, not a different mechanism.
**Trade-offs:** JudgmentTarget payloads become more complex. Consumers that don't use resolution candidates see nullable fields.
**Sources:** engine CLAUDE.md (Judgment Foundation Types, JudgmentTarget sections)
**Exploration:** quick
**Status:** captured

## D4: Retrieval feedback activation

**Choice:** Automatic via classpath-activated decorators; cbr: block is the opt-in gate
**Alternatives:**
- Separate feedbackConfig block on CaseDefinition — unnecessary YAML ceremony when the cbr: block already gates
- Always-on with opt-out — functionally equivalent but less explicit
**Rationale:** Neocortex decorators (CbrRetrievalTracker, RetrievalTracker) fire transparently when on the classpath. They only record when retrieval happens, which is already gated by the cbr: block. Engine's job is wiring feedback evaluation after worker completion.
**Trade-offs:** None significant — this is how neocortex decorators already work.
**Sources:** neocortex memory-cbr-tracking module, rag-tracking module
**Exploration:** quick
**Status:** captured

## D5: Mixed retrieval result type

**Choice:** Extend RetrievedExperience with nullable fields for document content
**Alternatives:**
- New sealed RetrievalResult type — type-safe but requires changing every consumer
- Separate retrieval, merge at presentation — internal code stays type-safe but duplicates ranking logic
**Rationale:** One list, one type, existing threading works. RetrievedExperience already flows through AgentRoutingContext, WorkerContext, EventLog metadata, and JudgmentTarget payloads.
**Trade-offs:** Nullable fields on RetrievedExperience — consumers must check sourceType.
**Sources:** engine CLAUDE.md (CBR Retrieval Bridge section, RetrievedExperience definition)
**Exploration:** quick
**Status:** captured

## D6: CBR naming cleanup

**Choice:** Prerequisite issue separate from #1081
**Alternatives:**
- First child issue of epic — groups CBR changes but delays the epic start
**Rationale:** Mechanical, cross-cutting, XS-scale. Blocks nothing in the spec. Land before the epic.
**Trade-offs:** One extra issue to track.
**Sources:** CbrRetrievalService.BUILT_IN_TYPES (engine runtime)
**Exploration:** quick
**Status:** captured

## D7: Outcome weighting default

**Choice:** Flip casehub.cbr.outcome-weighting.enabled to true + document
**Alternatives:**
- Per-case override field — more control, more YAML ceremony
- Leave as-is — consumers may never discover it
**Rationale:** Outcome weighting is universally beneficial when CBR is active. One property change, XS effort.
**Trade-offs:** Existing deployments that upgrade get outcome weighting automatically. Unlikely to cause issues — the weighting is a multiplier that favors higher-confidence cases.
**Sources:** neocortex OutcomeWeightingCbrCaseMemoryStore decorator
**Exploration:** quick
**Status:** captured

## D8: Document structure

**Choice:** Structured ResolutionStep records alongside free-form prose
**Alternatives:**
- Free-form prose only — LLMs parse it themselves; engine can't reason about steps
- Defer to ingestion adapter — no consistency across adapters
**Rationale:** Three consumption modes need three representations: plan trace (automated), structured steps (LLM-guided), prose (human-readable). The step structure enables the engine to track per-step outcomes and feed them back into CBR.
**Trade-offs:** Ingestion adapters must produce structured steps, which requires parsing documents.
**Sources:** engine#1081 issue (three consumption modes table)
**Exploration:** quick
**Depends on:** D2 (corpus source model)
**Status:** captured

## D9: Selection gate model

**Choice:** JudgmentTarget pre-dispatch gate for human selection
**Alternatives:**
- Routing strategy with human fallback — blurs routing and judgment
- Configuration-driven mode switch — too rigid, doesn't compose
**Rationale:** A judgment binding fires before the capability binding. It presents ranked candidates, the human selects, the selection writes to context, the capability binding fires with the selection. Automated path skips the judgment — agent selects directly via routing strategy.
**Trade-offs:** Requires two bindings (judgment + capability) for the human path. Automated path needs only the capability binding.
**Sources:** engine CLAUDE.md (Unified Judgment Scheduling, JudgmentTarget sections)
**Depends on:** D3 (HITL model)
**Exploration:** quick
**Status:** captured
