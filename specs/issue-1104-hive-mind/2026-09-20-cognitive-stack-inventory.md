# CaseHub Cognitive Stack — Capabilities Inventory

Comprehensive inventory of the blocks social cognition and neocortex memory subsystems, mapped for the cognitive self-improvement research document.

---

## 1. Cognition Core (Central Orchestrator)

### CognitionCore
**Package:** `io.casehub.blocks.agentic.social`
**Role:** Central orchestrator that composes all cognitive subsystems into a unified tick-based lifecycle. Each cognitive cycle (`tick()`) evaluates mood, drives, narrative, strategy, user models, mental models, goals, and memory hygiene. Also handles interaction recording with automatic mood appraisal (via LLM) and BDI extraction from conversation.

**Key methods:**
- `tick(agentId, tenantId, descriptor, resolver)` — runs one cognitive cycle across all enabled subsystems in order: mood → memoryHygiene → narrative → drives → strategy → userModel/mentalModel per subject → goals
- `recordInteraction(agentId, tenantId, subjectId, userMessage, response, impact)` — records an interaction, appraises mood via LLM prompt, extracts BDI from utterances
- `promptSections()` — assembles all cognitive state into prompt sections that shape agent behavior (personality, constraints, mood, drives, narrative, userModel, mentalModel, strategy, goals)

**Subsystems orchestrated:**
| Subsystem | Class | Required |
|-----------|-------|----------|
| Mood | `MoodOrchestrator` | Yes (always present) |
| Drives | `DriveOrchestrator` | Yes (always present) |
| User Model | `UserModelOrchestrator` | Optional |
| Mental Model | `MentalModelOrchestrator` | Optional |
| Strategy | `StrategyLearningOrchestrator` | Optional |
| Narrative | `NarrativeOrchestrator` | Optional |
| Goals | `GoalProposalOrchestrator` | Optional |
| Memory Hygiene | `MemoryHygieneOrchestrator` | Optional |

### CognitionConfig
**Role:** Feature-flag record controlling which subsystems are active. Enables selective cognition — a lightweight agent can run mood+drives only; a full cognitive agent runs everything.

**Key fields:** `moodEnabled`, `drivesEnabled`, `mentalModelEnabled`, `userModelEnabled`, `strategyEnabled`, `narrativeEnabled`, `goalsEnabled`, `memoryHygieneEnabled`, `directivePrompts`

**Factory methods:** `CognitionConfig.all()` (everything on), `CognitionConfig.none()` (everything off), `.with(subsystem, enabled)`, `.without(subsystems...)`

**Self-improvement relevance:** The self-improvement agent would use `CognitionConfig.all()` for full cognitive depth. Engine-only fallback uses `.none()` with rule-based budget allocation.

### CognitionSnapshot
**Role:** Point-in-time capture of the full cognitive state: mood, drives, mental models, user profiles, strategy, narrative, goal proposals. Used for diffing between turns.

**Key fields:** `mood (MoodState)`, `drives (DriveProfile)`, `mentalModels (Map<subjectId, MentalModelSnapshot>)`, `userProfiles (Map<subjectId, UserProfile>)`, `strategy (StrategyProfile)`, `narrative (NarrativeState)`, `goalProposals (List<DriveGoalProposal>)`

**Key methods:** `capture(core, agentId, tenantId, turnNumber, subjectIds)` — static factory; `diffFrom(previous)` — computes `CognitionDelta`

### CognitionDelta
**Role:** Computed difference between two snapshots. Tracks mood shifts (P/A/D deltas), drive intensity changes per axis, BDI changes per subject, user profile familiarity changes, new narrative episodes/themes, and new goal proposals.

**Sub-records:** `MoodDelta`, `DriveDelta`, `BdiDelta`, `ProfileDelta`

### CognitionMetrics
**Role:** Per-turn metrics combining prompt section count, delta, and full snapshot. Provides `summary()` and `toMarkdownRow()` for observability/logging.

---

## 2. Drive System (Needs/Motivation)

### DriveAxis (enum)
**Values:** `CURIOSITY`, `COMPETENCE`, `AFFILIATION`, `AUTONOMY`

**Self-improvement mapping:**
- CURIOSITY → research urge, knowledge gaps, exploring new techniques
- COMPETENCE → quality/stability concerns, declining performance metrics
- AFFILIATION → coordination quality, team cohesion, relationship health
- AUTONOMY → capability expansion, desire for greater self-determination

### DriveSource (functional interface)
**Contract:** `DriveIntensity evaluate(agentId, tenantId)` — evaluates one drive axis, returns intensity [0.0, 1.0] with trigger description.

**Self-improvement relevance:** The primary SPI for feeding self-improvement metrics into the drive system. Implement `DriveSource` for each axis with improvement-specific evaluation logic.

### DriveIntensity (record)
**Fields:** `axis (DriveAxis)`, `intensity (double 0-1)`, `trigger (String)` — a single measurement of one need's urgency, with the reason why.

### DriveComposer
**Role:** Composes raw drive intensities into a weighted `DriveProfile`, applying mood modulation (PAD), personality modulation (via AgentDisposition), and narrative modulation (via theme weights). Finds the dominant drive and computes composite motivation.

**Key methods:**
- `compose(rawDrives, disposition, mood, narrativeModulation, config, ...)` → `DriveProfile`
- `applyMoodModulation(intensity, axis, mood, config)` — arousal uniformly boosts intensity; pleasure boosts AFFILIATION/COMPETENCE more; low dominance boosts AUTONOMY drive
- `applyPersonalityModulation(intensity, axis, disposition, config)` — maps drive axes to disposition axes (CURIOSITY→RISK_APPETITE, COMPETENCE→RULE_FOLLOWING, AFFILIATION→SOCIAL_ORIENTATION, AUTONOMY→AUTONOMY)
- `computeComposite(drives, config)` — weighted average of all drive intensities
- `findDominant(drives)` — highest-intensity axis

**Self-improvement relevance:** This is WHERE mood and personality influence improvement priorities. A frustrated agent (low pleasure) naturally shifts toward competence work. An excited agent (high arousal) amplifies all drives.

### DriveOrchestrator
**Role:** Per-agent drive evaluation on each cognitive tick. Evaluates all four drive sources, feeds into DriveComposer, tracks changes above threshold, and maintains per-agent DriveProfile history.

**Key methods:**
- `tick(agentId, tenantId, descriptor)` → `DriveTick` (sealed: NoChange | Updated with changed axes)
- `currentDrives(agentId, tenantId)` → `Optional<DriveProfile>`

**Constructor wiring (production):** Takes `MemoryHygieneOrchestrator` → wraps in `CuriosityDrive`; `StrategyLearningOrchestrator` → wraps in `CompetenceDrive`; `UserModelOrchestrator` → wraps in `AffiliationDrive`; `MentalModelOrchestrator` → wraps in `AutonomyDrive`

### DriveConfig (record)
**Key fields:** `axisWeights (Map<DriveAxis, Double>)` — per-axis weight for composite calculation (defaults: all 1.0 — balanced); `changeThreshold` — minimum delta to report as a change; mood/personality modulation strengths; min/max intensity bounds; `narrativeModulationStrength`

**Self-improvement relevance:** Configurable axis weights allow the self-improvement agent to emphasize certain needs. E.g., higher COMPETENCE weight makes it more stability-focused; higher CURIOSITY weight makes it more research-oriented.

### DriveProfile (record)
**Fields:** `agentId`, `tenantId`, `drives (Map<DriveAxis, DriveIntensity>)`, `compositeMotivation (0-1)`, `dominantDrive (DriveAxis)`, `evaluatedAt`

### DriveTick (sealed interface)
**Variants:** `NoChange(reason)`, `Updated(previous, current, changedAxes)`

### Concrete Drive Sources

**CuriosityDrive:** Evaluates knowledge gaps via `MemoryHygieneOrchestrator.knowledgeGaps()`. Intensity = average of (lowRetentionRatio, consolidationGroupDiversity). High when lots of under-reinforced memories across many groups.

**CompetenceDrive:** Evaluates engagement trends via `StrategyLearningOrchestrator.engagementTrend()`. Intensity = declining dimension ratio minus improvement discount. High when many engagement dimensions are declining.

**AffiliationDrive:** Evaluates relationship health via `UserModelOrchestrator.activeProfiles()`. Intensity = ratio of neglected relationships (low familiarity or stale). High when relationships are decaying.

**AutonomyDrive:** Evaluates mental model pressure via `MentalModelOrchestrator.activeSnapshots()`. Intensity = accumulated high-confidence intentions from tracked subjects. High when many subjects have strong, unaddressed intentions.

### DriveAlignment (record, emergence package)
**Role:** Measures alignment of drive profiles across a set of agents. Per-axis alignment scores + composite alignment + dominant shared axis.

**Self-improvement relevance:** In a swarm, measures whether agents share the same improvement motivations. High alignment → coordinated improvement focus.

---

## 3. Mood System (PAD Emotions)

### MoodState (record, neocortex memory-api)
**Package:** `io.casehub.neocortex.memory.mood`
**Fields:** `agentId`, `tenantId`, `timestamp`, `pleasure [-1,1]`, `arousal [-1,1]`, `dominance [-1,1]`, `cause (String)`, `turnId`, `activeContextIds`, `metadata`

**Self-improvement mapping:**
- **Pleasure:** rises on successful improvements (PR merged, tests green), drops on failures (PR rejected, regression detected)
- **Arousal:** rises on discovery (promising paper, new technique), spikes on crisis (stability failure), drops during routine maintenance
- **Dominance:** rises when capabilities expand, drops when hitting capability walls or receiving critical feedback

### MoodOrchestrator
**Role:** Manages per-agent emotional state with signal-based updates and exponential decay toward baseline. Tick-based processing: drains pending signals, applies deltas, clamps to max displacement, applies time-based decay toward baseline.

**Key methods:**
- `record(signal, agentId, tenantId)` — queues a mood signal (direct shift or interaction appraisal)
- `tick(agentId, tenantId)` → `MoodTick` (NoChange | Updated with new state, signal count, whether decay occurred)
- `currentMood(agentId, tenantId)` → `Optional<MoodState>`

**Decay model:** Exponential decay toward `MoodBaseline` with configurable `decayTimeConstant`. Emotions are transient unless continuously reinforced — a frustration from a failed PR naturally fades unless reinforced by more failures.

**CognitionCore integration:** Mood appraisal via LLM — each interaction is appraised for P/A/D delta by asking the LLM to rate the emotional tone, clamped to [-0.3, +0.3] per axis.

### MoodPromptSection
**Role:** Renders current emotional state into a prompt section that influences agent behavior. Interprets P/A/D values as labels: pleasure → positive/negative/neutral, arousal → energetic/calm/balanced, dominance → confident/submissive/balanced.

---

## 4. Narrative System (Inner Monologue)

### NarrativeOrchestrator
**Role:** Manages the agent's inner narrative — a synthesised account of its experience structured as episodes (individual and group) and derived themes. Loaded from `NarrativeStore` on each tick; tracks what's changed between synthesis cycles.

**Key methods:**
- `tick(agentId, tenantId)` → `NarrativeTick` (NoChange | Updated with new episodes and themes)
- `currentNarrative(agentId, tenantId)` → `Optional<NarrativeState>`

### NarrativeState (record)
**Fields:** `scopeId`, `tenantId`, `scope (NarrativeScope)`, `fragments (List<NarrativeFragment>)`, `synthesisedAt`, `reflectionCountAtSynthesis`

**Key methods:** `episodes()` → individual episodes; `groupEpisodes()` → group episodes; `themes()` → derived themes; `dominantTheme()` → highest-salience theme

**Fragment types (sealed hierarchy):**
- `IndividualEpisode` — a single agent's narrative arc
- `GroupEpisode` — shared experience across agents
- `DerivedTheme` — recurring pattern with salience score and axis modulation weights

### NarrativeModulation
**Role:** Converts narrative themes into drive modulation weights. Each theme has per-axis weights and a salience score; the modulation is `salience × weight` per axis, accumulated across themes.

**Self-improvement relevance:** If the narrative develops a persistent theme like "we keep struggling with test coverage," that theme modulates the COMPETENCE drive upward through salience × weight. The agent's inner story directly shapes what it prioritises.

---

## 5. Strategy Learning (What Approaches Work)

### StrategyLearningOrchestrator
**Role:** Metacognitive strategy advisor. Tracks engagement signals (turn outcomes, conversation outcomes), stores CBR cases for past interactions, analyses trends, and uses LLM + reflection to periodically update strategy profiles with ranked guidelines and dimensional adjustments.

**Key methods:**
- `record(signal, agentId, subjectId, tenantId)` — records engagement signals (TurnOutcome, ConversationOutcome)
- `tick(agentId, tenantId)` → `StrategyLearningTick` (NoChange | Observed | Learned with CBR cases stored)
- `reflect(agentId, tenantId)` → `StrategyReflection` (NoChange | Reflected with updated profile, guidelines, trends)
- `currentStrategy(agentId, tenantId)` → `Optional<StrategyProfile>`
- `engagementTrend(agentId, tenantId)` → `Optional<EngagementTrend>`

**Strategy dimensions (default 5):** verbosity, formality, initiative, directness, questionRate — each a float [0, 1] that adjusts the agent's communication approach.

**Learning loop:** Records interaction features as CBR cases → analyses trends (TrendAnalyzer with slope, delta, volatility) → retrieves past cases + reflections → LLM synthesises guidelines + dimensional deltas → applies updates.

**Self-improvement relevance:** Feeds CompetenceDrive (engagement trends directly measure how well the agent is performing). The strategy profile accumulates learned guidelines about what improvement approaches work.

---

## 6. Mental Model (Understanding of Others/Environment)

### MentalModelOrchestrator
**Role:** Maintains per-subject BDI (Belief-Desire-Intention) models with confidence scores, entrenchment (repeated reinforcement), half-life decay, and LLM-based inference. Models what the agent believes about each subject's mental state.

**Key methods:**
- `record(signal, agentId, subjectId, tenantId)` — records verbal cues classified as BELIEF_STATEMENT, DESIRE_EXPRESSION, or INTENTION_DECLARATION
- `tick(agentId, subjectId, tenantId)` → `MentalModelTick` (Unchanged | Updated | Inferred)
- `project(agentId, subjectId, tenantId)` → `List<MentalProjection>` — high-confidence attributions
- `activeSnapshots(agentId, tenantId)` → all tracked subjects' models
- `observeConversation(commonGround, ...)` — integrates common ground facts as beliefs

**BDI lifecycle:** Heuristic extraction → signal buffering → optional LLM inference → confidence decay (half-life per dimension) → eviction below floor → snapshot persistence.

**Self-improvement relevance:** Feeds AutonomyDrive (intention pressure from subjects). For self-improvement, the "subjects" include human reviewers (what do they want from PRs?), other agents (what are their improvement goals?), and the platform itself (what does it need?).

### UserModelOrchestrator
**Role:** Maintains behavioral profiles per interaction partner: communication style, topics of interest, preferences, relationship stage (based on familiarity score). LLM-based synthesis updates profile when sufficient signals accumulate.

**Key methods:**
- `record(signal, agentId, subjectId, tenantId)` — records positive/negative/neutral interaction signals
- `tick(agentId, subjectId, tenantId)` → `UserModelTick` (Unchanged | Updated | Synthesised)
- `currentProfile(agentId, subjectId, tenantId)` → `UserProfile`
- `activeProfiles(agentId, tenantId)` → all tracked subject profiles

**Familiarity model:** `computeFamiliarity(positive, negative, neutral, stageConfig, ticksSinceLastInteraction)` — affect-weighted volume with time decay. Higher familiarity = deeper relationship.

**Self-improvement relevance:** Feeds AffiliationDrive. The self-improvement agent builds relationships with human reviewers and other agents — tracking communication preferences, response patterns, and what kind of PR descriptions each reviewer prefers.

---

## 7. Goal System (Drive-to-Goal Bridge)

### GoalProposalOrchestrator
**Role:** Bridge between drive intensity and concrete goal proposals. When a drive exceeds threshold, proposes a goal via mappers or LLM formation strategy. Also evaluates existing goals for abandonment (failure threshold, staleness) and priority adjustments (narrative-driven escalation/demotion).

**Key methods:**
- `tick(agentId, tenantId, descriptor)` → `GoalProposalTick` (NoChange | Changes with proposals, abandonments, priority adjustments, governance updates)
- `registerGoals(agentId, tenantId, goals)` — manually register goals
- `currentProposals(agentId, tenantId)` → `Optional<List<DriveGoalProposal>>`

**Goal lifecycle managed:**
1. **Formation:** Drive intensity ≥ `proposalThreshold` → DriveGoalMapper or DriveGoalFormationStrategy produces a proposal
2. **Escalation:** Narrative themes aligned with a goal for `escalationCycles` synthesis rounds → goal escalated to PRIMARY priority
3. **Demotion:** Previously escalated goal loses narrative alignment for `demotionCycles` rounds → demoted back to SECONDARY
4. **Abandonment:** Goal's originating drive drops below `relevanceThreshold` for `staleAfter` duration OR `failureAbandonmentThreshold` failures → abandoned
5. **Cross-axis compounds:** Narrative themes spanning multiple drive axes generate compound goals (e.g., COMPETENCE + CURIOSITY)

### DriveGoalMapper (functional interface)
**Contract:** `@Nullable DriveGoalProposal evaluate(agentId, tenantId, intensity)` — maps a drive intensity to a concrete goal proposal. Multiple mappers can be registered; first non-null wins.

### DriveGoalFormationStrategy (functional interface)
**Contract:** `@Nullable DriveGoalProposal propose(DriveGoalFormationContext context)` — LLM-backed goal formation. Context includes existing goals and remaining capacity.

### DriveGoalProposal (record)
**Fields:** `axis`, `goalName`, `goalDescription`, `formationReason`, `driveIntensity`, `suggestedPriority`, `proposalAttributes`

### DriveGoalFormationContext (record)
**Fields:** `agentId`, `tenantId`, `axis`, `intensity`, `trigger`, `existingGoals`, `remainingCapacity`

**Self-improvement relevance:** This is the direct bridge from "I need to improve" (drive) to "here's what I'll do" (goal). The self-improvement agent's DriveGoalMappers produce SELF_IMPROVEMENT goals from improvement-related drive intensity. Narrative escalation promotes recurring improvement themes to PRIMARY goals.

---

## 8. Memory Architecture (Episodic, Semantic, Consolidation)

### MoodState (neocortex memory-api)
See section 3 above. Persisted as part of the memory substrate.

### MindMap (neocortex mindmap)
**Components:** `MindMapStore`, `MindMapNode`, `MindMapEdge`, `MindMapSubgraph`
**Role:** Semantic knowledge graph with typed nodes and edges, organized into subgraphs. Nodes carry emotional valence (`pleasure`, `arousal`, `dominance` fields on MindMapNode), temporal bounds (`validFrom`, `validUntil`), confidence scores, and metadata. Edges are typed relationships (e.g., `APPLIES_TO`, `CAUSES`, `CONTRADICTS`).

### CuriositySignalGenerator (neocortex mindmap-intelligence)
**Role:** Generates curiosity signals from knowledge graph analysis. Six signal categories drive exploration:
- **STRUCTURAL:** orphan nodes (disconnected concepts), sparse subgraphs
- **QUALITY:** contradictions, low-confidence clusters, unvalidated edges
- **TEMPORAL:** stale nodes (not updated recently), past events without recorded outcomes
- **CENTRALITY:** high betweenness centrality (bridging concepts), high degree centrality (hub concepts)
- **PROXIMITY:** approaching events (future dates), past events needing closure

**Modulation:** Affect dampening (negative-valence nodes get curiosity boost; worsening affect trajectory amplifies), topical distance dampening (signals near recent conversation topics score higher via BFS distance).

**Self-improvement relevance:** The CuriositySignalGenerator would identify knowledge gaps in the self-improvement agent's understanding of the platform: orphan concepts (techniques not yet connected to platform modules), contradictions (conflicting improvement outcomes), stale knowledge (outdated platform understanding), high-centrality concepts (core abstractions worth understanding deeply).

### CuriositySignal (record)
**Fields:** `category (SignalCategory)`, `score (0-1)`, `targetNodeId`, `targetSubgraphId`, `question (natural language)`, `description`

### ExperienceConsolidationPhase (neocortex mindmap-intelligence)
**Role:** Promotes episodic buffer events to semantic MindMap nodes via a graduation scoring pipeline. Events that pass a threshold (importance score based on graduation context) become persistent knowledge nodes.

**Consolidation lifecycle:**
1. Scan episodic buffer for ungraduated events
2. Score each event via `GraduationScorer` (customizable scoring)
3. Classify via `GraduationClassifier` (what type of MindMap node to create)
4. Create/update MindMap nodes for events exceeding graduation threshold
5. Mark events as graduated (via cursor tracking)

**Self-improvement relevance:** This is how ephemeral improvement experiences (a build broke, a PR was reviewed, a paper was read) become permanent knowledge. The graduation threshold controls what's worth remembering permanently — high-impact events (significant regressions, breakthrough improvements) graduate; routine maintenance may not.

### ConsolidationPhase (interface)
**Contract:** `name()`, `run(tenantId, subgraphPriority)` — extensible consolidation pipeline. Multiple phases run in priority order.

### KnowledgeGapSummary (record)
**Fields:** `lowRetentionCount`, `consolidationGroups`, `totalScored`
**Role:** Summary of memory hygiene state — how many memories have low retention, how many consolidation groups need attention. Feeds into CuriosityDrive intensity.

---

## 9. Prompt Integration (How Cognitive State Becomes Agent Behavior)

### PromptSection (interface, blocks speech)
**Contract:** `@Nullable String contribute(PromptContext)` — returns text to inject into the agent's system prompt, or null if nothing to contribute.

### Concrete Prompt Sections

| Section | Source | What it renders |
|---------|--------|----------------|
| `PersonalityPromptSection` | `AgentDescriptor.disposition()` | Personality profile text |
| `ConstraintPromptSection` | `AgentDescriptor.constraints()` | Behavioral constraints |
| `MoodPromptSection` | `MoodOrchestrator` | P/A/D values with interpretive labels |
| `DrivePromptSection` | `DriveOrchestrator` | Motivational state via `CognitiveObservationSections` |
| `NarrativePromptSection` | `NarrativeOrchestrator` | Current inner narrative |
| `UserModelPromptSection` | `UserModelOrchestrator` | Relationship profiles |
| `MentalModelPromptSection` | `MentalModelOrchestrator` | BDI attributions |
| `StrategyPromptSection` | `StrategyLearningOrchestrator` | Strategy guidelines and dimensional values |
| `GoalPromptSection` | `GoalProposalOrchestrator` | Active goal proposals |

**DirectiveSection wrapper:** When `CognitionConfig.directivePrompts` is true, each section is wrapped in `DirectiveSection` which transforms observational prompts into directive/instructional framing.

**Self-improvement relevance:** The cognitive state becomes the agent's behavior through these prompt sections. The self-improvement agent's prompt would include its current emotional state, active drives, narrative about what it's been working on, learned strategy guidelines, and goal proposals — all shaping how the LLM reasons about its next improvement action.

---

## 10. Integration Points for Self-Improvement

### Primary SPI: DriveSource
Implement four `DriveSource` instances feeding improvement-specific metrics into the drive system:
- **ImprovementCuriositySource:** Research opportunity signals, unexamined techniques, knowledge gaps in platform understanding
- **ImprovementCompetenceSource:** CI status, test pass rate, lint violation count, coverage percentage, dependency staleness
- **ImprovementAutonomySource:** Problem classes with repeated failures, trust score plateaus, capability gaps
- **ImprovementAffiliationSource:** Team coherence (from swarm TeamDetector), coordination failure rate, reviewer relationship health

### Bridge: DriveGoalMapper / DriveGoalFormationStrategy
Map improvement drive intensity to concrete SELF_IMPROVEMENT goals:
- COMPETENCE drive high → propose stability/quality improvement goal
- CURIOSITY drive high → propose research/exploration goal
- AUTONOMY drive high → propose capability expansion goal
- AFFILIATION drive high → propose coordination improvement goal

### Narrative → Priority
`NarrativeModulation` feeds back from narrative themes to drive intensity. Persistent improvement themes ("test coverage keeps being a problem") naturally escalate improvement goals to PRIMARY priority through the GoalProposalOrchestrator's escalation mechanism.

### MindMap → Memory
Store all improvement experiences as MindMap nodes with emotional associations. Research findings, build outcomes, PR review feedback, agent coordination events — all become semantic knowledge that persists across sessions and consolidates over time.

### CuriositySignalGenerator → Research Direction
The curiosity signal generator identifies knowledge gaps that direct research: orphan concepts (techniques not connected to implementation), contradictions (conflicting outcome data), stale knowledge (outdated platform understanding). These feed into the CURIOSITY drive source.

### Outcome → Mood
Improvement outcomes feed PAD signals: successful PR → positive pleasure; rejected PR → negative pleasure + negative dominance; discovered promising research → positive arousal; regression detected → negative pleasure + positive arousal (alert).

### Full Loop
```
Platform metrics → DriveSource evaluations → DriveOrchestrator.tick()
    → DriveComposer (modulated by MoodState + AgentDisposition + NarrativeModulation)
    → DriveProfile (dominant drive emerges)
    → GoalProposalOrchestrator.tick()
    → DriveGoalMapper produces SELF_IMPROVEMENT goal
    → Goal dispatched as improvement case
    → Improvement case lifecycle (introspect → research → implement → review → integrate)
    → Outcome → MoodSignal (emotional response) + EventLog + CBR trace + MindMap node
    → Mood shifts → DriveComposer modulation changes → next tick's priorities shift
```

This is a closed cognitive loop — experiences shape emotions, emotions modulate drives, drives propose goals, goals produce actions, actions create outcomes, outcomes become experiences.
