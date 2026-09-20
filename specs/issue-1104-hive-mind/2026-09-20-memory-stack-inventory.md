# Neocortex + Blocks Cognitive Memory Stack — Capabilities Inventory

Comprehensive mapping of the CaseHub cognitive architecture. All classes are shipping code unless noted.

---

## 1. MindMap / Semantic Knowledge Graph

The MindMap is a typed semantic graph with PAD emotional dimensions on every node and edge.

### Core API (`mindmap-api`)
| Class | Role |
|-------|------|
| `MindMapNode` | Graph node — `id`, `name`, `subgraphType`, `traits`, `refs`, `properties`, `confidence`, `provenance`, temporal validity (`validFrom`/`validUntil`), **PAD emotional dimensions** (`pleasure()`, `arousal()`, `dominance()`) directly on the interface, `principalId` for ownership, `sharedWith` for visibility |
| `MindMapEdge` | Typed directed edge — `sourceNodeId`, `targetNodeId`, `edgeType`, `ValidationTier`, `confidence`, `provenance`, temporal validity, **PAD emotional dimensions** on every edge, properties |
| `MindMapStore` | Full graph store SPI — `addNode`, `addEdge`, `updateNode`, `removeEdge`, `addAlias`, `resolveNode`, `mergeNodes`, subgraph CRUD, `neighbors`, `bridgeEdges`, `search`, `supersede`/`reinstate`, `eraseNode`/`eraseSubgraph`/`eraseEntity`, batch ops (`addNodes`/`addEdges`) |
| `MindMapSubgraph` | Partitioned subgraph — nodes grouped by semantic type |
| `MindMapQuery` | Query record for search |
| `MindMapVocabulary` | Registered edge types, node types, trait definitions |
| `MindMapCapability` | Enum: `TRAVERSAL`, `MERGE`, `VOCABULARY`, `ALIAS`, `SUBGRAPH`, `SEARCH`, `SUPERSESSION`, `ERASE_NODE`, `ERASE_SUBGRAPH`, `ERASE_ENTITY`, `CROSS_TENANT_ERASE`, `GRAPH_ANALYSIS` |
| `MindMapConfidenceDefaults` | Default confidence values by origin |
| `NodeInput`, `EdgeInput`, `NodeUpdate`, `SubgraphInput` | Builder records for mutations |
| `NodeRef` | Cross-reference to external entities |

### Key insight: **PAD is first-class on every node AND edge.** Every piece of knowledge and every relationship in the graph carries emotional valence. This is not metadata — it's structural. A memory node about a failed improvement attempt carries negative pleasure. A research paper node carries excitement (high arousal). Retrieval can be biased by emotional congruence with current mood.

### Intelligence layer (`mindmap-intelligence`)
| Class | Role |
|-------|------|
| `MindMapExtractor` | LLM-powered entity/relationship extraction from text → nodes + edges |
| `ExtractedRelationship` / `ParsedRelationship` | Intermediate extraction results |

### Analysis (`mindmap-core`)
| Class | Role |
|-------|------|
| `MindMapAnalyzer` | Graph analysis: betweenness centrality, subgraph structure, isolated nodes |

### Storage backends
- `InMemoryMindMapStore` (`mindmap-inmem`) — testing
- `SqliteMindMapStore` (`mindmap-sqlite`) — lightweight persistent
- `NoOpMindMapStore` — fallback when no backend configured

---

## 2. Episodic Memory & Experience Events

Sealed hierarchy of typed experience events that record what an agent perceives, does, and outcomes.

### Core API (`memory-api`)
| Class | Role |
|-------|------|
| `ExperienceEvent` | Sealed interface: `Observation`, `Action`, `Outcome`. Fields: `agentId`, `tenantId`, `caseId`, `turnId`, `timestamp`, `description`, `confidence`, `metadata` |
| `Observation` | What the agent noticed — has additional `subject` field |
| `Action` | What the agent did |
| `Outcome` | What resulted from the action |
| `ExperienceEvents` | Domain registration and factory (`DOMAIN = "experience"`) |
| `ExperienceAttributeKeys` | Standard metadata keys for experience memories |

### Key insight: The sealed hierarchy enforces a perceive→act→outcome cycle. Every experience is typed, timestamped, and attributable to an agent in a specific case and turn.

---

## 3. Memory Consolidation (Episodic → Semantic Promotion)

Bio-inspired consolidation promotes ephemeral experience to durable knowledge, analogous to sleep-based memory consolidation.

### Consolidation framework (`mindmap-intelligence`)
| Class | Role |
|-------|------|
| `ConsolidationPhase` | SPI — `name()`, `run(tenantId, subgraphPriority)`, `beginTick()`. Phases execute in priority order. |
| `ConsolidationScheduler` | Orchestrates consolidation phases, manages scheduling and tick lifecycle |
| `ConsolidationCompleted` | Event record fired after consolidation run |
| `ConsolidationAuditEntry` | Audit trail for consolidation operations (cognitive-observability) |
| `ExperienceConsolidationPhase` | Core phase: graduates episodic memories to MindMap nodes. Uses `GraduationScorer` to score memories and `GraduationClassifier` to classify them. Configurable `threshold` (default 0.5), `maxPerPass` (default 20), `minCorroboration` (default 3 — requires multiple observations before promoting). Cursor-based for incremental processing. |
| `CuriosityRefreshPhase` | Phase that refreshes curiosity signals after consolidation |
| `ExperienceConsolidationConfig` | `threshold`, `maxPerPass`, `minCorroboration` |

### Key insight: `minCorroboration = 3` means a single observation doesn't become knowledge. Three independent observations corroborating the same fact are required. This prevents noise from becoming belief. Directly relevant for self-improvement: a single test failure doesn't trigger an improvement — consensus is needed.

---

## 4. CBR (Case-Based Reasoning)

Structured similarity-based retrieval for "find similar past situations."

### Core API (`memory-api`)
| Class | Role |
|-------|------|
| `CbrCaseRetriever` | Retrieval SPI — query with feature vector, returns scored matches |
| `CbrCaseMemoryStore` | Store for CBR cases (extends `CaseMemoryStore`) |
| `CbrCase` / `FeatureVectorCbrCase` | Case record with typed feature vectors |
| `CbrQuery` | Query with feature vector, weights, scope |
| `FeatureValue` | Typed feature value (numeric, categorical, text, vector) |
| `ScoredCbrCase` | Result with similarity score |
| `TemporalDecay` | Time-based decay for case relevance |
| `ScopeDecay` | Scope-based decay for case relevance |

### RAG pipeline (`rag-api`, `rag-core`)
| Class | Role |
|-------|------|
| `CaseRetriever` | Base retrieval SPI |
| `HybridCaseRetriever` | Combines vector + keyword search |
| `PayloadBoostCaseRetriever` | Metadata-based boosting |
| `RerankingCaseRetriever` | Cross-encoder reranking |
| `CorrectiveCaseRetriever` | CRAG — corrective retrieval with fallback |
| `QueryExpandingCaseRetriever` | Query augmentation for better recall |
| `TrackingCaseRetriever` | Retrieval analytics and tracking |

### Key insight: CBR provides structured "find similar" while MindMap provides relational "navigate and understand." Both are projections of the same agent experience, serving different query patterns.

---

## 5. Relationship Memory

Per-agent-pair interaction tracking with quality signals.

### Core API (`memory-api`)
| Class | Role |
|-------|------|
| `RelationshipEvent` | Inter-agent interaction record — `agentId`, `otherAgentId`, `sourceEventType`, `QualitySignal` (POSITIVE/NEGATIVE/NEUTRAL), `description`, `confidence` |
| `QualitySignal` | Enum: `POSITIVE`, `NEGATIVE`, `NEUTRAL` |
| `RelationshipEvents` | Domain registration (`DOMAIN = "relationship"`) |
| `RelationshipQuery` | Query builder for relationship history |
| `RelationshipRecorded` | Event fired after relationship event stored |

### Blocks extensions
| Class | Role |
|-------|------|
| `RelationshipStageConfig` | Configurable relationship stages with thresholds |
| `RelationshipPressureSource` | Pressure signal from relationship dynamics |
| `RelationshipSignal` | Inter-agent interaction signal (sealed member of `InteractionSignal`) |
| `RelationshipCue` | Mental model cue from relationships (sealed member of `MentalStateSignal`) |
| `RelationshipProcessor` | Core processor for relationship events (`memory-core`) |

### Key insight: Relationship memory feeds the AFFILIATION drive via `AffiliationDrive`. When relationships decay (low familiarity, stale interactions), affiliation drive rises. For self-improvement: the agent tracks relationships with human reviewers, other agents, and teams — quality of those relationships influences improvement decisions.

---

## 6. Curiosity & Knowledge Gap Detection

Curiosity signals emerge from knowledge gaps in the MindMap, driving exploration.

### Core classes (`mindmap-intelligence`)
| Class | Role |
|-------|------|
| `CuriositySignal` | Record: `SignalCategory`, `score`, `targetNodeId`, `targetSubgraphId`, `question`, `description` |
| `CuriositySignalGenerator` | `@ApplicationScoped` — scans MindMap for knowledge gaps: isolated nodes, thin edges, referenced-but-unexplored concepts. Uses `MindMapAnalyzer` + `CaseMemoryStore` + affect trajectory analysis. |
| `CuriositySignalProvider` | SPI interface for alternative curiosity signal sources |
| `CuriosityRefreshPhase` | Consolidation phase that refreshes curiosity signals after graph changes |
| `CuriosityConfig` | Configuration: thresholds, max signals, decay |

### Blocks integration
| Class | Role |
|-------|------|
| `CuriosityDrive` | `DriveSource` implementation — feeds CURIOSITY axis from `MemoryHygieneOrchestrator.knowledgeGaps()`. Intensity from low-retention memory count and consolidation group diversity. |
| `CuriosityGoalMapper` | Maps curiosity drive intensity to goal proposals |

### Key insight: Curiosity is bidirectional — from the MindMap (structural gaps in the knowledge graph) AND from memory hygiene (low-retention memories that need reinforcement). For self-improvement: "I found a reference to technique X but haven't explored it" generates a curiosity signal that feeds the CURIOSITY drive, which proposes a research goal.

---

## 7. Modulation Factors (Retrieval Biasing)

Pluggable factors that bias memory retrieval based on cognitive state.

### Core API (`cognitive-api`)
| Class | Role |
|-------|------|
| `ModulationFactor<T>` | Functional interface — `double apply(T item, ModulationProfile<T> profile)`. Biases retrieval scoring. |
| `ModulationProfile<T>` | Projection functions: `confidence`, `pleasure`, `arousal`, `dominance`, `timestamp` — extracts PAD and metadata from any memory type for modulation |

### Built-in factors (`cognitive-index`)
| Class | Role |
|-------|------|
| `ModulationFactors.recencyDecay(halfLife, now)` | Exponential time-based decay |
| `ModulationFactors.confidenceWeight()` | Weight by confidence value |
| `ModulationFactors.moodCongruence(mood, influence)` | **Mood-congruent retrieval** — biases toward memories whose PAD matches current mood state. Distance in PAD space → alignment score. |
| `ModulationFactors.domainWeight(personalityWeights)` | Personality-weighted domain bias |
| `ModulationContext` | Context record for modulation pipeline |
| `ModulationProfiles` | Factory for common profiles |

### Key insight: `moodCongruence` is the bridge between emotion and memory. When the agent is frustrated (low pleasure), retrieval biases toward memories with similar emotional valence — reinforcing the frustration pattern but also surfacing relevant failure memories. When excited (high arousal), retrieval biases toward exciting discoveries. This creates mood-congruent cognition — how you feel shapes what you remember.

---

## 8. Engagement & Interaction Tracking

Tracks how agents engage with each other and with subjects — response quality, affect shifts, interaction patterns.

### Core API (`memory-api`)
| Class | Role |
|-------|------|
| `EngagementEvent` | Detailed interaction record — `agentId`, `otherAgentId`, `responded`, `responseTimeMs`, `responseLength`, `affectShift` [-1,1], `reactionCount`, `continued` |
| `EngagementEvents` | Domain registration (`DOMAIN = "engagement"`) |
| `EngagementRecorded` | Event fired after storage |
| `EngagementRecorderCore` | Core processor (`memory-core`) |

### Blocks extensions
| Class | Role |
|-------|------|
| `EngagementTrend` | Trend analysis across engagement dimensions with `TrendDirection` (IMPROVING/DECLINING/STABLE) |
| `EngagementSignal` | Sealed interface for engagement-derived signals |

### Key insight: `EngagementEvent.affectShift` directly captures how an interaction changed the agent's emotional state. For self-improvement: when a human reviews a PR and gives harsh feedback, the `affectShift` is negative — this feeds MoodOrchestrator (PAD pleasure drops) → DriveComposer modulates drives → the agent becomes more cautious with that reviewer's domain.

---

## 9. Cognitive Node Types (Beliefs, Intentions, Predictions, etc.)

Nodes in the MindMap are classified by cognitive type via traits.

### Trait system (`mindmap-api`)
- Nodes carry `Set<String> traits()` — open vocabulary of cognitive classifications
- Issue #322 (closed) added classification: beliefs, intentions, fears, judgments, predictions
- Traits are applied by `DeclarativeTraitRule` in `CognitiveDefaults`

### Blocks mental model (`blocks-core`)
| Class | Role |
|-------|------|
| `MentalModelOrchestrator` | Tracks BDI (Belief-Desire-Intention) models per-subject |
| `MentalModelSnapshot` | Current state: `beliefs`, `desires`, `intentions` — each with confidence |
| `MentalStateSignal` | Signals from mental model changes (sealed: `BeliefChange`, `DesireChange`, `IntentionChange`, `RelationshipCue`) |

### Blocks cognition (`blocks-core`)
| Class | Role |
|-------|------|
| `CognitionCore` | Central orchestrator — composes: MoodOrchestrator, DriveOrchestrator, UserModelOrchestrator, MentalModelOrchestrator, StrategyLearningOrchestrator, NarrativeOrchestrator, GoalProposalOrchestrator, MemoryHygieneOrchestrator |
| `CognitionSnapshot` | Point-in-time capture of complete cognitive state: mood, drives, mental models, user profiles, strategy, narrative, goal proposals |
| `CognitionDelta` | Diff between snapshots: `MoodDelta` (PAD changes), `DriveDelta` (intensity changes), `BdiDelta` (belief/desire/intention count changes), `ProfileDelta`, episode/theme counts, new goal proposals |
| `CognitionPhase` | Evaluation ordering: `FOUNDATION` → `SOURCE` → `SOURCE_PER_SUBJECT` → `DERIVED` → `TERMINAL` |
| `CognitionConfig` | Feature flags: mood, drives, mentalModel, userModel, strategy, narrative, goals, memoryHygiene, innerLife, directivePrompts |
| `CognitionMetrics` | Cognitive system metrics |
| `CognitionTickContext` | Per-tick context for cognition evaluation |
| `CognitionTickParticipant` | SPI for custom cognition tick participants |

### Key insight: `CognitionConfig` has `innerLifeEnabled` — the agent has inner life (internal narrative, self-reflection). `CognitionPhase.TERMINAL` is where side-effects happen (goal proposals, strategy updates). The cognitive pipeline is deterministic and ordered.

---

## 10. Memory Hygiene & Maintenance

Ongoing memory health management — knowledge gap detection, retention scoring, cross-linking, consolidation.

### Blocks (`blocks-core`)
| Class | Role |
|-------|------|
| `MemoryHygieneOrchestrator` | Manages memory health: retention scoring (`ConfidenceScorer`), temporal decay, scope decay, consolidation batching, cross-link discovery, knowledge gap detection. Produces `KnowledgeGapSummary` consumed by `CuriosityDrive`. |
| `MemoryHygieneScheduler` | Schedules periodic hygiene ticks |
| `KnowledgeGapSummary` | Summary of knowledge gaps: `totalScored`, `lowRetentionCount`, `consolidationGroups` |
| `ConfidenceScorer` | Scores memory retention quality |
| `RetentionConfig` | Configuration for retention thresholds |
| `HygieneEvent` | Sealed events: `ReflectionGenerated`, etc. |

### Reflection subsystem (`memory-api`, `memory-core`, `blocks-core`)
| Class | Role |
|-------|------|
| `ReflectionEvent` | Higher-order insight synthesized from experience — `insight`, `level` (depth of reflection), `sourceMemoryIds` (provenance to raw memories), `confidence` |
| `ReflectionOrchestrator` | SPI for triggering reflection |
| `ReflectionSynthesizer` | SPI for LLM-backed insight extraction from experience trajectories |
| `ReflectionStore` / `ReflectionQueryStore` | Storage and retrieval for reflections |
| `ReflectionEntry` | Persistent reflection with metadata |
| `ReflectionEventAdapter` | Adapts reflections into narrative fragments |

### Key insight: `ReflectionEvent.level` supports recursive reflection — reflection on reflections. Level 1: "I noticed test failures in module X." Level 2: "I keep noticing test failures in module X — there might be a systemic issue." Level 3: "My tendency to focus on module X failures is because it's where I had my worst experience. I should look at the data more objectively." This is meta-cognition.

---

## Cross-Cutting: The Drive System (Motivation)

### Blocks (`blocks-core`, `io.casehub.blocks.agentic.social.drive`)
| Class | Role |
|-------|------|
| `DriveAxis` | Enum: `CURIOSITY`, `COMPETENCE`, `AFFILIATION`, `AUTONOMY` |
| `DriveSource` | Functional SPI: `DriveIntensity evaluate(agentId, tenantId)` |
| `DriveIntensity` | Record: axis, intensity [0,1], trigger description |
| `DriveComposer` | Composes raw drives with mood modulation (PAD), personality modulation (disposition axes), and narrative modulation. Produces weighted `DriveProfile` with `dominantDrive`. |
| `DriveOrchestrator` | Per-agent orchestration: evaluates all 4 axes each tick, composes via `DriveComposer`, computes `DriveTick` (changed axes above threshold). |
| `DriveProfile` | Result: per-axis intensities, composite motivation [0,1], dominant drive |
| `DriveTick` | Sealed: `NoChange` or `Updated(previous, current, changedAxes)` |
| `DriveConfig` | Weights per axis, change threshold, mood/personality modulation strengths, min/max intensity |

### Built-in drive sources:
| Class | Source | Intensity from |
|-------|--------|---------------|
| `CuriosityDrive` | `MemoryHygieneOrchestrator` | Low-retention memories, consolidation group diversity |
| `CompetenceDrive` | `StrategyLearningOrchestrator` | Declining engagement dimensions |
| `AffiliationDrive` | `UserModelOrchestrator` | Neglected relationships (low familiarity, stale interactions) |
| `AutonomyDrive` | `MentalModelOrchestrator` | High-confidence intentions across subjects |

### Goal formation from drives:
| Class | Role |
|-------|------|
| `GoalProposalOrchestrator` | Orchestrates drive→goal pipeline with escalation, cross-axis enrichment, narrative context |
| `DriveGoalFormationStrategy` | SPI: proposes goals from drive context |
| `DriveGoalFormationContext` | Context: axis, intensity, trigger, existing goals, remaining capacity |
| `DriveGoalProposal` | Proposed goal: axis, name, description, reason, intensity, priority, attributes |
| `DriveGoalMapper` | Maps drive intensity to goal proposals |
| `GoalEscalationPolicy` | Escalation when drives exceed thresholds |
| `CrossAxisGoalEnricher` | Enriches proposals with cross-axis context |

---

## Cross-Cutting: Narrative (Inner Thoughts)

### Blocks (`blocks-core`, `io.casehub.blocks.agentic.social.narrative`)
| Class | Role |
|-------|------|
| `NarrativeOrchestrator` | Manages the agent's inner narrative across scopes |
| `NarrativeState` | Current narrative: fragments (episodes + themes), synthesis timestamp |
| `NarrativeFragment` | Sealed: `IndividualEpisode`, `GroupEpisode`, `DerivedTheme` |
| `NarrativeScope` | Scope of narrative synthesis |
| `NarrativeModulation` | Computes drive modulation from narrative state |
| `NarrativePipeline` | Processing pipeline for narrative synthesis |
| `NarrativeSynthesisGate` | Gating control for synthesis timing |
| `NarrativeConfig` | Configuration for narrative synthesis |

### Key insight: Narrative modulates drives via `NarrativeModulation`. The agent's inner story ("I've been struggling with this module") shifts drive intensities, which shifts goal proposals. The narrative is not just reporting — it's a cognitive force that shapes behaviour.

---

## Cross-Cutting: Mood (PAD Emotional State)

### Neocortex (`memory-api`)
| Class | Role |
|-------|------|
| `MoodState` | Record: `agentId`, `tenantId`, `pleasure` [-1,1], `arousal` [-1,1], `dominance` [-1,1], `cause`, `turnId`, `activeContextIds`, `metadata` |
| `MoodEvents` | Domain: `DOMAIN = "mood"` |
| `AffectEvents` | Domain: `DOMAIN = "affect"` — affect trajectory tracking |

### Blocks (`blocks-core`)
| Class | Role |
|-------|------|
| `MoodOrchestrator` | Manages PAD state per agent: updates from interaction signals, decays toward baseline |
| `MoodBaseline` | Per-agent resting PAD values (part of `CognitiveDefaults`) |
| `AffectTrajectoryAnalyzer` | Tracks emotional trajectory over time (cognitive-index) |

### Key insight: MoodState carries a mandatory `cause` field — every emotional state change has a documented reason. For self-improvement: "improvement:ci-failure" causes arousal spike; "improvement:pr-approved" causes pleasure increase. The cause chain is auditable.

---

## Summary: How It All Connects

```
Experience (Observation/Action/Outcome)
    │
    ├─→ CaseMemoryStore (episodic buffer)
    │       │
    │       ├─→ MoodOrchestrator (PAD shift from experience)
    │       │       │
    │       │       └─→ DriveComposer (mood modulates drives)
    │       │
    │       ├─→ ExperienceConsolidationPhase (promote to MindMap)
    │       │       │
    │       │       └─→ MindMap nodes (with PAD from experience)
    │       │               │
    │       │               ├─→ CuriositySignalGenerator (knowledge gaps)
    │       │               │       │
    │       │               │       └─→ CuriosityDrive (CURIOSITY axis)
    │       │               │
    │       │               └─→ ModulationFactors.moodCongruence (retrieval bias)
    │       │
    │       ├─→ EngagementEvents (interaction quality)
    │       │       │
    │       │       └─→ CompetenceDrive (COMPETENCE axis)
    │       │
    │       ├─→ RelationshipEvents (inter-agent quality)
    │       │       │
    │       │       └─→ AffiliationDrive (AFFILIATION axis)
    │       │
    │       └─→ ReflectionEvents (meta-cognition)
    │               │
    │               └─→ NarrativeOrchestrator (inner story)
    │                       │
    │                       └─→ NarrativeModulation (narrative modulates drives)
    │
    └─→ CBR traces (structured outcome retrieval)

DriveOrchestrator.tick()
    │
    ├─→ 4 DriveSource evaluations
    ├─→ DriveComposer (mood + personality + narrative modulation)
    ├─→ DriveProfile (dominant drive, composite motivation)
    └─→ GoalProposalOrchestrator (drive → goal proposals)

CognitionCore.tick()
    │
    ├─→ All of the above, ordered by CognitionPhase
    ├─→ CognitionSnapshot (complete cognitive state)
    └─→ CognitionDelta (what changed)
```
