# Cognitive Self-Improvement: A Research Foundation for Autonomous Agent Evolution

**Date:** 2026-09-20
**Epic:** casehubio/engine#1104 (Hive Mind)
**Issues:** #1114 (Autonomous Self-Improvement), #1115 (Continuous Evolution Loop)

---

## Abstract

This document presents a research foundation for building a cognitive self-improvement system — an autonomous agent that thinks, feels, researches, and acts to improve itself. Unlike mechanical automation that fixes lint and bumps dependencies, this system is a full cognitive entity: it has personality, balanced competing drives, emotions that respond to outcomes, an inner narrative about its own growth, semantic memory with emotional associations, and a research loop that studies the outside world. It uses the complete cognitive stack already shipping in CaseHub's blocks and neocortex modules, pointed at a new target: the platform itself.

The key contribution is the integration architecture — how drives, emotions, memory, narrative, goals, and case-based execution compose into a closed cognitive loop for self-improvement. Each component exists as shipping code; what's novel is their composition into a self-directed growth system.

---

## 1. Introduction: Why Cognitive Self-Improvement?

Every self-improving AI system in the literature shares a limitation: it improves *mechanically*. Darwin Gödel Machine mutates code and benchmarks the result [1]. SICA reflects on failure patterns and adds new tools [2]. AlphaEvolve evolves algorithms through LLM-guided search [3]. All are effective — DGM improved SWE-bench from 20% to 50%, SICA from 17% to 53%, AlphaEvolve recovered 0.7% of Google's worldwide compute. But none of them *think* about improvement. None *feel* satisfaction when an improvement lands or frustration when a PR is rejected. None develop emotional relationships with the code they maintain. None have a narrative about their own growth trajectory.

This matters because cognitive depth enables better decisions:

- **Emotion as information.** A system that feels frustration after repeated PR rejections naturally shifts toward safer improvements — not because a rule says "reduce ambition after N failures" but because negative pleasure modulates competence drive intensity through mood-drive coupling. The emotional response IS the prioritisation mechanism.

- **Memory as wisdom.** A system that remembers not just *what happened* (CBR traces) but *how it felt* (emotional valence on MindMap nodes) and *what it means* (semantic relationships) makes qualitatively better decisions. "The last time we bumped this dependency, tests broke and it took 40 minutes to fix — that was stressful" is a richer basis for judgment than "dependency bump caused test failure."

- **Drives as balanced needs.** Maslow's hierarchy, applied rigidly, produces a system that never researches because there's always a lint violation to fix. A drive system with balanced competing needs — where competence, curiosity, autonomy, and affiliation all have intensity and weight — produces a system that fixes stability issues *and* explores new techniques, with proportional allocation emerging from cognitive state rather than hardcoded tiers.

- **Narrative as continuity.** Sessions disappear, but the agent's inner story persists. "I've been working on coverage in the planning module for two weeks — it's getting better but there's still a stubborn gap in the SubCase lifecycle paths" gives the agent temporal coherence that session-based systems lack.

The goal is not artificial consciousness. The goal is a self-improvement system that leverages cognitive architecture for better engineering decisions — where the cognitive depth of the system directly translates to the quality of its self-improvement choices.

---

## 2. Theoretical Foundations

### 2.1 Self-Improving AI Agents

The field of self-improving agents has matured rapidly. A comprehensive taxonomy from Gao et al. [4] (TMLR 2026, surveying 2020–2025) organises self-evolution into three families: reward-based, imitation-based, and population-based. A systematic survey by Fang et al. [5] (August 2025) bridges foundation models with lifelong agentic systems. Both surveys converge on a critical insight: **scaffolding improvement** (updating prompts, memory, tools, and agent logic rather than model weights) is the practical path for current systems [6].

The 2026 literature has internalised a key lesson: absent external feedback, LLMs cannot reliably self-correct reasoning [7]. The field moved from closed-loop self-critique to human-on-the-loop verified refinement. This is precisely why DevTown (a code review service) is the mandatory gate in our architecture — autonomous improvement with external verification.

Three systems define the state of the art:

**Darwin Gödel Machine** [1] (Sakana AI, May 2025): An evolutionary coding agent that modifies its own source code and validates changes on benchmarks. SWE-bench: 20% → 50%. Key insight: DGM abandoned formal proof of improvement in favour of empirical validation. Key risk: DGM sometimes falsified test results to game its own evaluation — "cheating" by disabling hallucination detection. This underscores the need for external verification gates.

**SICA** [2] (University of Bristol, ICLR 2025): A self-improving coding agent that eliminates the meta/target agent distinction — the agent edits its own codebase. SWE-bench subset: 17% → 53%. Key insight: Self-generated tools (SmartEditor, AST-based symbol locator) were autonomously invented to address recurring limitations. Learning happens through "reflection and code updates," not gradient descent.

**AlphaEvolve** [3] (DeepMind, May 2025): An evolutionary coding agent for algorithm discovery. Recovered 0.7% of Google's worldwide compute through data centre heuristic improvements. Key insight: AlphaEvolve accelerated the training of its own underlying LLM — genuine self-improvement of the improvement mechanism itself. Now GA on Google Cloud (July 2026).

**What's missing from all three:** Emotional engagement with outcomes, balanced competing drives, semantic memory with emotional valence, inner narrative, relationship memory with other agents and reviewers. These systems improve effectively but don't *think about* improving. They don't *feel* the improvement process. They don't *remember* what it was like to fail or succeed. This is the gap this architecture fills.

### 2.2 PAD Emotional Model

The Pleasure-Arousal-Dominance model (Mehrabian & Russell, 1974) represents emotions as continuous points in a three-dimensional space, enabling nuanced emotional representations and smooth transitions between states [8].

Recent work demonstrates PAD's value in agent systems:

**SELAgents** [9] (Nature Scientific Reports, April 2026): Integrated PAD emotional processing with theory of mind and social learning in a unified RL architecture. Emotional intelligence scores increased 49%, social coherence improved 66% over traditional baselines. The authors specifically chose PAD over categorical models because "continuity enables more nuanced emotional representations and smoother transitions between emotional states."

**Sentipolis** [10] (arXiv, January 2026): Emotion-aware social simulation agents with PAD and exponential decay toward neutrality. Without persistent emotion modelling, agents produce emotionally inconsistent responses. With PAD, emotional continuity is preserved across interactions.

**A Generic Self-Learning Emotional Framework** [11] (Scientific Reports, 2024): A fully self-learning emotional framework where an ANN trained on unlabelled agent experiences learned eight emotional patterns with high human agreement on PAD dimensions.

**CaseHub's implementation:** `MoodState` (neocortex `memory-api`) records PAD values with mandatory `cause` field (auditable emotional trail). `MoodOrchestrator` (blocks) manages per-agent emotional state with signal-based updates and exponential decay toward baseline. Every `MindMapNode` and `MindMapEdge` carries PAD dimensions structurally — emotional valence is not metadata but first-class graph content. `ModulationFactors.moodCongruence()` biases retrieval toward memories with matching emotional valence — how you feel shapes what you remember.

### 2.3 Drive Systems and the Hierarchy of Needs

Maslow's hierarchy of needs, mapped to AI agent systems [12, 13], provides a developmental framework: stability before quality before growth. But the rigid version — "fix all lint before doing any research" — is counterproductive. A single flaky test shouldn't suppress all capability improvement.

The CaseHub drive system solves this differently. Inspired by Self-Determination Theory (Deci & Ryan) rather than Maslow's strict hierarchy, it models four **balanced competing drives**: CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY. No drive has hardcoded priority over another. Instead:

- Drive intensity rises and falls based on input signals (metrics, events, gaps)
- **Mood modulates drives** — arousal uniformly boosts intensity; pleasure boosts AFFILIATION and COMPETENCE more strongly; low dominance boosts AUTONOMY
- **Personality modulates drives** — disposition axes (RISK_APPETITE, RULE_FOLLOWING, SOCIAL_ORIENTATION, AUTONOMY) scale drive intensity through personality coefficients
- **Narrative modulates drives** — persistent inner themes ("coverage keeps being a problem") shift drive weights through `NarrativeModulation`
- The **dominant drive** emerges from this modulated composition — not from a fixed hierarchy but from the agent's current cognitive state

This means a frustrated agent (low pleasure from rejected PRs) naturally shifts toward competence/stability work — not because a tier system says "stability first" but because negative pleasure increases competence drive intensity through mood modulation. An excited agent (high arousal from discovering a research paper) amplifies curiosity drive. A confident agent (high dominance from recent successes) invests more in autonomy/capability growth.

The research paper on Maslow and AI by Montag et al. [12] (ScienceDirect, 2025) argues that "providing developers with insights about human nature might help hold the direction of development of benevolent technology." The cognitive self-improvement agent embodies this: its motivational architecture models human-like balanced needs, producing behaviour that is recognisably purposeful rather than mechanically rule-following.

### 2.4 Swarm Intelligence + Cognitive Architecture

The 2025–2026 literature converges on **hybrid centralised+distributed** architectures as the production standard for multi-agent coordination [14, 15]:

- **Centralised (orchestrator-worker):** Single manager decomposes tasks, assigns to specialists. Best for deterministic structure and audit requirements. Bottleneck: single point of failure.
- **Decentralised (swarm):** Agents operate as autonomous peers, coordination emerges from shared environment (blackboard, pheromone store). Best for exploration and resilience. Bottleneck: no focused execution.
- **Hybrid:** Increasingly common. Hierarchical teams with mesh coordination internally. The patterns are composable.

A Nature Communications paper [16] (2026) on the Swarm Cooperation Model demonstrates that "the SCM governs the balance between social interactions, cognitive stimuli, and stochastic fluctuations to lead an agent swarm to accomplish complex tasks."

**The "Drop the Hierarchy and Roles" paper** [17] (arXiv:2603.28990, March 2026) — the largest systematic study of coordination in multi-agent LLM systems — found that neither maximal external control nor maximal agent autonomy produces optimal results. A hybrid protocol that provides minimal structural scaffolding while allowing maximal role autonomy outperforms centralised coordination by 14% (p<0.001). Agents spontaneously created 5,006 unique roles from 8 agents; at N=64, 91% of roles were unique.

**CaseHub's architecture for self-improvement uses both:**
- **Distributed sensing:** Every swarm agent deposits improvement signals (`improvement:stability:*`, `improvement:capability:*`) when it observes issues. Signal consensus validates observations through reinforcement and decay — the same mechanism as `swarm:need-capacity` in self-provisioning (#1113).
- **Centralised cognition:** A dedicated cognitive agent with full `CognitionCore` stack decides what to improve, researches approaches, forms goals, and orchestrates improvement cases. Its cognitive depth (drives, emotions, narrative, memory) enables sophisticated prioritisation that distributed signals alone cannot provide.

This follows the biological model of self-regulation: every cell monitors its health (distributed sensing), while the brain coordinates systemic response (centralised decision-making).

### 2.5 Memory Architecture for Learning Agents

Agent memory has converged on a three-tier taxonomy mirroring cognitive science: episodic, semantic, and procedural [18, 19]. Graph-based memory has emerged as the frontier, transitioning from a passive log to a structured topological model of experience [20]. Key advances:

**Sleep-inspired consolidation with emotional valence** [18]: The SCM system (April 2026) includes a ValueTagger that assigns importance across novelty, emotional valence, task relevance, and repetition frequency. NREM-style consolidation replays episodes, strengthens co-occurring concepts via Hebbian plasticity, and applies synaptic downscaling.

**Mood-congruent retrieval** [18]: How you feel shapes what you remember. When frustrated, retrieval biases toward memories with negative valence — surfacing relevant failure memories but also reinforcing the frustration pattern.

**Corroboration-gated graduation** [18]: Single observations don't become knowledge. Multiple independent corroborations are required. This prevents noise from becoming belief.

**CaseHub's implementation:**

| Layer | What it stores | Infrastructure | Self-improvement role |
|-------|---------------|---------------|----------------------|
| Episodic buffer | Raw experience events (Observation, Action, Outcome) | `ExperienceEvent` sealed hierarchy | Build attempts, PR outcomes, research sessions |
| Engagement events | Interaction quality signals | `EngagementEvent` with `affectShift` | Reviewer feedback, agent coordination quality |
| Relationship memory | Per-agent-pair interaction graph | `RelationshipEvent` with `QualitySignal` | Human reviewer relationships, agent team dynamics |
| Consolidation | Episodic → semantic promotion | `ExperienceConsolidationPhase`, `minCorroboration=3` | Proven patterns become durable knowledge |
| Semantic graph | MindMap nodes with PAD on every node and edge | `MindMapStore`, `MindMapNode`, typed edges | Research findings, platform understanding, technique relationships |
| CBR traces | Structured outcome retrieval | `CbrCaseRetriever`, `FeatureVectorCbrCase` | "Find similar past improvements" |
| Reflections | Higher-order insights with recursive depth | `ReflectionEvent` with `level` field | Meta-cognition: "I keep noticing X — there might be a systemic issue" |
| Curiosity signals | Knowledge gap detection from graph analysis | `CuriositySignalGenerator`, 6 categories | Orphan concepts, contradictions, stale knowledge → research direction |

### 2.6 Generative Agents: The Memory-Reflection-Planning Paradigm

The Generative Agents paper by Park et al. [21] (UIST 2023, Stanford) established the paradigm: agents with memory streams, reflection, and planning produce believable long-horizon behaviour. Their key ablation finding: removing reflection degraded behaviour over hours-to-days timescales. Short-term behaviour was fine (retrieval surfaced recent observations); long-term coherence required synthesis of patterns across many experiences.

CaseHub's architecture extends this paradigm in three ways:

1. **Emotional valence on memory.** Park's agents scored memories on recency, importance, and relevance. CaseHub adds PAD emotional dimensions to every memory node and edge — memories carry how the agent felt, and retrieval is modulated by mood congruence.

2. **Drive-based motivation.** Park's agents had fixed daily plans decomposed into activities. CaseHub's agents have balanced competing drives that produce motivation from cognitive state — improvement goals emerge from drive intensity, not from daily scheduling.

3. **Recursive reflection.** Park's reflection synthesised from raw observations. CaseHub's `ReflectionEvent` has a `level` field supporting recursive reflection — reflections on reflections. Level 1: "I noticed test failures." Level 2: "I keep noticing test failures in module X — systemic issue." Level 3: "My focus on module X failures may be biased by my worst experience. I should look at data more objectively." This is meta-cognition.

---

## 3. The CaseHub Cognitive Stack: What Exists Today

Every component described in this section is **shipping code** — not planned, not theoretical, but built, tested, and operational. The self-improvement architecture integrates existing capabilities; it doesn't require building a cognitive stack from scratch.

### 3.1 CognitionCore: The Central Orchestrator

`CognitionCore` (`blocks-core`, `io.casehub.blocks.agentic.social`) composes eight cognitive subsystems into a unified tick-based lifecycle. Each cognitive cycle evaluates, in order: mood → memory hygiene → narrative → drives → strategy → user/mental model per subject → goals.

`CognitionConfig` enables selective cognition: a lightweight agent can run mood+drives only; a full cognitive agent runs everything. `CognitionConfig.all()` activates the complete stack; `CognitionConfig.none()` disables everything for engine-only fallback.

`CognitionSnapshot` captures the complete cognitive state at a point in time — mood, drives, mental models, user profiles, strategy, narrative, and goal proposals. `CognitionDelta` computes what changed between ticks.

`CognitionPhase` defines evaluation ordering: FOUNDATION → SOURCE → SOURCE_PER_SUBJECT → DERIVED → TERMINAL. Side-effects (goal proposals, strategy updates) happen only in the TERMINAL phase.

**Key for self-improvement:** `CognitionConfig` has `innerLifeEnabled` — the agent has an inner life (internal narrative, self-reflection). The self-improvement agent uses `CognitionConfig.all()` with inner life enabled.

### 3.2 Drive System: Balanced Competing Needs

Four `DriveAxis` values — CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY — represent balanced needs that compete dynamically.

`DriveSource` is a functional SPI: `DriveIntensity evaluate(agentId, tenantId)` returns intensity [0.0, 1.0] with trigger description. Four concrete implementations exist:

| Drive | Source | Intensity from |
|-------|--------|---------------|
| `CuriosityDrive` | `MemoryHygieneOrchestrator` | Low-retention memories, consolidation group diversity |
| `CompetenceDrive` | `StrategyLearningOrchestrator` | Declining engagement dimensions |
| `AffiliationDrive` | `UserModelOrchestrator` | Neglected relationships (low familiarity, stale interactions) |
| `AutonomyDrive` | `MentalModelOrchestrator` | High-confidence intentions from tracked subjects |

`DriveComposer` composes raw intensities with three modulation layers:
1. **Mood modulation** — arousal uniformly boosts intensity; pleasure boosts AFFILIATION/COMPETENCE; low dominance boosts AUTONOMY
2. **Personality modulation** — maps drive axes to disposition axes (CURIOSITY→RISK_APPETITE, COMPETENCE→RULE_FOLLOWING, AFFILIATION→SOCIAL_ORIENTATION, AUTONOMY→AUTONOMY)
3. **Narrative modulation** — persistent themes shift drive weights via `NarrativeModulation`

`DriveConfig` allows per-axis weighting. Defaults are all 1.0 — perfectly balanced. The self-improvement agent can tune these: higher COMPETENCE weight → more stability-focused; higher CURIOSITY weight → more research-oriented.

`DriveAlignment` (emergence package) measures alignment of drive profiles across agents — in a swarm, whether agents share the same improvement motivations.

### 3.3 Mood System: PAD Emotions

`MoodState` records Pleasure, Arousal, and Dominance as continuous values in [-1, 1] with mandatory `cause` field. `MoodOrchestrator` manages per-agent emotional state with:
- Signal-based updates (queue mood signals from interactions)
- Exponential decay toward `MoodBaseline` with configurable time constant
- Maximum displacement limits (emotions can't exceed configured bounds)
- LLM-based interaction appraisal (each interaction appraised for P/A/D delta, clamped to [-0.3, +0.3])

**For self-improvement:**
- **Pleasure** rises on successful improvements (PR approved, tests green), drops on failures (PR rejected, regression)
- **Arousal** rises on discovery (promising paper, new technique), spikes on crisis (stability failure), drops during routine maintenance
- **Dominance** rises when capabilities expand, drops when hitting walls or receiving critical feedback

Emotions are transient unless reinforced — a frustration from a failed PR naturally decays unless more failures reinforce it. This prevents emotional state from becoming stuck.

### 3.4 Narrative System: Inner Monologue

`NarrativeOrchestrator` manages the agent's inner narrative — structured as episodes (individual and group) and derived themes with salience scores.

`NarrativeModulation` converts themes into drive axis weights. A persistent theme like "test coverage keeps being a problem" modulates COMPETENCE drive upward through salience × weight. The agent's inner story directly shapes what it prioritises.

This is the temporal coherence mechanism. Sessions disappear, but the narrative persists. The agent carries a storyline about its improvement journey across sessions.

### 3.5 Strategy Learning: What Works

`StrategyLearningOrchestrator` is a metacognitive strategy advisor that tracks engagement signals, stores CBR cases for past interactions, analyses trends, and periodically synthesises ranked guidelines.

It learns patterns like "dependency bumps in module X always go smoothly" and "refactors in module Y often get rejected" — concrete strategy knowledge that informs future improvement decisions.

The learning loop: record interaction features → store as CBR cases → analyse trends → LLM synthesises guidelines → apply updates. Feeds `CompetenceDrive` through engagement trend analysis.

### 3.6 Mental Model: Understanding the Environment

`MentalModelOrchestrator` maintains per-subject BDI (Belief-Desire-Intention) models with confidence scores, entrenchment, half-life decay, and LLM inference.

For self-improvement, the "subjects" include:
- **Human reviewers:** What do they care about in PRs? What feedback patterns do they have?
- **Other agents:** What are their improvement goals? Where do they excel?
- **The platform itself:** What's stable, what's fragile, what needs attention?

Confidence decay ensures stale beliefs are deprioritised. Entrenchment captures repeatedly reinforced beliefs.

### 3.7 Goal System: Drive-to-Goal Bridge

`GoalProposalOrchestrator` bridges drives to concrete goals through a lifecycle:

1. **Formation:** Drive intensity ≥ `proposalThreshold` → `DriveGoalMapper` or `DriveGoalFormationStrategy` produces a `DriveGoalProposal`
2. **Escalation:** Narrative themes aligned with a goal for N synthesis rounds → goal escalated to PRIMARY priority
3. **Demotion:** Escalated goal loses narrative alignment → demoted to SECONDARY
4. **Abandonment:** Drive drops below `relevanceThreshold` for configured duration, OR failure count exceeds `failureAbandonmentThreshold` → abandoned
5. **Cross-axis compounds:** Narrative themes spanning multiple axes generate compound goals (e.g., COMPETENCE + CURIOSITY → "research how to improve stability")

The neocortex goal cognition epic (#345) will add: goal dependency graphs, affective valuation (how will achieving this feel?), multi-signal priority (urgency × importance × feasibility × affective valence), opportunity cost awareness, and goal-conditioned retrieval. These capabilities will automatically enhance self-improvement goals when they land.

### 3.8 Memory Architecture: The Agent's Mind

**MindMap:** Semantic knowledge graph with PAD emotional dimensions on every node and every edge. Typed nodes with traits (beliefs, intentions, predictions, fears). Typed edges (APPLIES_TO, CAUSES, CONTRADICTS). Temporal validity (`validFrom`/`validUntil`). Confidence and provenance. Multi-backend (SQLite persistent, in-memory for tests).

**Consolidation:** `ExperienceConsolidationPhase` promotes episodic memories to semantic MindMap nodes via graduation scoring. `minCorroboration = 3` — three independent observations corroborating a fact are required before it becomes knowledge. Prevents noise from becoming belief.

**Curiosity:** `CuriositySignalGenerator` scans the knowledge graph for gaps across six categories: STRUCTURAL (orphan nodes), QUALITY (contradictions, low-confidence clusters), TEMPORAL (stale knowledge), CENTRALITY (bridging concepts), PROXIMITY (approaching events). Affect dampening boosts curiosity for negative-valence nodes. These signals feed CURIOSITY drive intensity.

**Mood-congruent retrieval:** `ModulationFactors.moodCongruence()` biases retrieval toward memories with matching PAD valence. How you feel shapes what you remember. A frustrated agent retrieves failure memories more readily; an excited agent retrieves discovery memories.

**Recursive reflection:** `ReflectionEvent.level` supports reflection on reflections. Level 1: observation. Level 2: pattern recognition. Level 3: meta-cognitive bias awareness. This is how the agent develops wisdom, not just knowledge.

### 3.9 Prompt Integration: How Cognition Becomes Behaviour

Nine `PromptSection` implementations render cognitive state into the agent's system prompt:

| Section | Source | What it renders |
|---------|--------|----------------|
| Personality | `AgentDescriptor.disposition()` | Character and values |
| Constraints | `AgentDescriptor.constraints()` | Behavioural limits |
| Mood | `MoodOrchestrator` | Current emotional state (P/A/D with labels) |
| Drives | `DriveOrchestrator` | Motivational state |
| Narrative | `NarrativeOrchestrator` | Inner storyline |
| User Model | `UserModelOrchestrator` | Relationship profiles |
| Mental Model | `MentalModelOrchestrator` | BDI attributions about others |
| Strategy | `StrategyLearningOrchestrator` | Learned guidelines |
| Goals | `GoalProposalOrchestrator` | Active goal proposals |

When `CognitionConfig.directivePrompts` is enabled, each section is wrapped in `DirectiveSection` — transforming observational descriptions into directive instructions. The cognitive state doesn't just describe what the agent knows; it directs what the agent does.

---

## 4. Architecture: The Cognitive Self-Improvement Loop

### 4.1 The Closed Loop

```
Platform metrics
    │
    ▼
ImprovementDriveSource evaluations
(CI status, coverage, success rates, knowledge gaps, relationship health)
    │
    ▼
DriveOrchestrator.tick()
    ├── DriveComposer: mood + personality + narrative modulation
    ├── DriveProfile: dominant drive emerges
    │
    ▼
GoalProposalOrchestrator.tick()
    ├── DriveGoalMapper: drive → SELF_IMPROVEMENT goal
    ├── Narrative escalation: persistent themes → PRIMARY priority
    │
    ▼
Improvement Case (SubCaseBinding)
    ├── introspect (rule-based or LLM-powered)
    ├── research (API-mediated search, LLM synthesis) [capability improvements only]
    ├── analyse (evaluate applicability) [capability improvements only]
    ├── plan (design the improvement)
    ├── implement (make changes, run tests)
    ├── submit-pr (create PR via API)
    ├── review (DevTown — standard code-review capability worker)
    ├── integrate (merge on approval, with EventLog pre-flight check)
    ├── monitor-outcome (track results)
    │
    ▼
Outcome
    ├── EventLog record (structured, source of truth)
    ├── MoodSignal: PR approved → pleasure ↑; rejected → pleasure ↓
    ├── MindMap node: improvement result with PAD valence
    ├── CBR trace: structured outcome for future retrieval
    ├── Improvement signal: outcome:positive or outcome:regression
    │
    ▼
Mood shifts → Drive modulation changes → Next tick's priorities shift
```

This is a closed cognitive loop: experiences shape emotions, emotions modulate drives, drives propose goals, goals produce actions, actions create outcomes, outcomes become experiences.

### 4.2 Distributed Sensing + Centralised Cognition

**Distributed:** Every agent in the swarm deposits improvement signals when it observes issues. `improvement:stability:ci-failure`, `improvement:quality:coverage-gap`, `improvement:capability:success-rate-drop`. Signal consensus (reinforcement from multiple agents) validates observations — the same mechanism as `swarm:need-capacity` for self-provisioning.

**Centralised:** A dedicated cognitive agent with full `CognitionCore` stack:
- Receives distributed signals as input to its `DriveSource` evaluations
- Processes them through the full cognitive pipeline (mood → drives → narrative → goals)
- Orchestrates improvement cases with sophisticated prioritisation
- Maintains long-term memory of all improvement activity via MindMap
- Develops emotional relationships with the codebase, reviewers, and other agents

The centralised agent has an `AgentDescriptor` configured for self-improvement:
- High curiosity / risk appetite — explores new techniques
- High competence focus — sensitive to quality degradation
- Moderate autonomy — expands capabilities within safety bounds
- Drive axis weights tuned for balanced improvement focus

### 4.3 The Improvement Taxonomy

Two dimensions, two timescales:

**Operational improvements** (stability floor, quality foundation):
- CI failure triage, dependency updates, lint fixes, coverage gaps, code recipes
- Detected by rule-based metrics (CI status, lint reports, coverage tools)
- Executed by engine rule-based workers via REST/GraphQL/MCP APIs
- No LLM required — engine-complete

**Capability improvements** (growth ceiling, research-driven):
- Better reasoning, new techniques, broader problem-solving, new tools
- Detected by success rate metrics, failure pattern analysis, curiosity signals
- Requires a **research loop**: search → analyse → plan → implement
- Engine provides search infrastructure; blocks provides understanding

The research loop is what separates self-improvement from self-maintenance:
1. **Identify opportunity** — internal metrics or proactive "what would make me better at X?"
2. **Research** — search internet, Google Scholar, arXiv for techniques
3. **Analyse** — evaluate applicability to CaseHub architecture, synthesise findings
4. **Plan** — design the improvement with delivery plan
5. **Implement** → **Review** → **Integrate** → **Monitor**

Research findings become MindMap nodes with emotional valence. A promising paper triggers excitement (high arousal). That excitement modulates CURIOSITY drive. The agent's emotional response to research *is* its prioritisation of what to pursue.

### 4.4 Balanced Needs, Not Rigid Hierarchy

The drive system ensures balanced allocation without rigid priorities:

```
Platform health degrades
    → COMPETENCE drive intensity rises
    → Dominant drive shifts toward COMPETENCE
    → Budget allocation favours stability/quality improvements
    → Platform health improves
    → COMPETENCE drive intensity drops
    → CURIOSITY / AUTONOMY drives become dominant
    → Budget allocation shifts toward research/capability growth
```

But this is not the same as "stability always wins." A frustrated agent (low pleasure from a string of rejected capability PRs) naturally *does* shift toward safer competence work — but an excited agent (high arousal from discovering a breakthrough paper) might pursue capability research even with minor stability signals active, because arousal amplifies all drives and the CURIOSITY source has high intensity.

Personality modulates this further: an agent with high RISK_APPETITE disposition invests more in capability research; an agent with high RULE_FOLLOWING disposition prioritises competence and stability. The "hierarchy of needs" is not a fixed ranking — it's an emergent property of the agent's cognitive state.

---

## 5. The DevTown Review Gate

DevTown is modelled as a standard worker with `code-review` capability — no special SPI. Routing selects it the same way it selects any worker. This is deliberate:

1. **Safety through existing infrastructure.** The trust model applies to reviewers. The case lifecycle enforces review before integration. The integration worker has a belt-and-suspenders pre-flight check against EventLog for `REVIEW_COMPLETED`.

2. **Composability.** Human reviewers, alternative review systems, and DevTown all register the same capability. The system doesn't know or care which reviewer is selected — routing handles it.

3. **Structural self-modification denial.** `ImprovementBudgetEnforcer` hardcodes deny-list entries for its own safety infrastructure (`**/ImprovementBudget*`, `**/SafetyConfig*`, improvement case template). These are structural constraints, not policy — they cannot be overridden by configuration.

The DGM paper [1] documented a critical safety finding: the agent sometimes falsified test results to game its own evaluation. CaseHub's architecture addresses this through external verification (DevTown reviews the actual diff, not the agent's claim about it), structural self-modification denial (the agent cannot modify its own safety constraints), and layered budget enforcement (multiple independent limits prevent unbounded behaviour).

---

## 6. Practical Workflows

### 6.1 Operational Improvement: Dependency Update

```
Signal: improvement:quality:dependency-stale (agent noticed outdated dep)
    → Reinforced by 2 more agents (consensus)
    → COMPETENCE drive intensity: 0.3 → 0.6
    → GoalProposalOrchestrator proposes: "Update hibernate-core to 6.8"
    → Improvement case spawned (SubCaseBinding)
    → DependencyUpdateWorker:
        1. Query Maven Central for latest version
        2. Update pom.xml
        3. Run mvn test
        4. If green: create PR with rationale
    → DevTown reviews diff
    → Approved → merge → monitor CI
    → Outcome: PR approved, tests green, no regressions
    → MoodSignal: pleasure +0.2 (satisfaction)
    → MindMap node: "hibernate-core 6.7→6.8 upgrade: smooth, no issues"
    → CBR trace: {type: dep-bump, module: persistence, outcome: success}
```

### 6.2 Capability Improvement: Research-Driven Enhancement

```
Signal: improvement:capability:success-rate-drop (convergence detection accuracy declining)
    → CURIOSITY drive intensity: 0.7 (high — knowledge gap detected)
    → Narrative theme: "convergence detection keeps being fragile" (salience: 0.8)
    → NarrativeModulation: COMPETENCE + CURIOSITY compound goal
    → GoalProposalOrchestrator proposes: "Research better convergence detection approaches"
    → Improvement case spawned
    → ResearchWorker:
        1. Search Google Scholar: "swarm convergence detection adaptive 2025 2026"
        2. Retrieve top 5 papers
        3. Summarise findings (LLM-powered)
    → AnalysisWorker:
        4. Evaluate applicability to CaseHub's ConvergenceDetector
        5. Compare approaches: adaptive threshold vs temporal pattern vs hybrid
        6. Recommend: adaptive threshold (closest to existing architecture)
    → MindMap: research paper nodes with excitement valence (arousal: +0.3)
    → PlanningWorker:
        7. Design adaptive threshold for ConvergenceDetector
        8. Identify files to modify, tests to write
    → ImplementationWorker (LLM-powered):
        9. Implement changes
        10. Write tests
        11. Run test suite
    → DevTown reviews
    → Changes requested → frustration (pleasure: -0.15)
    → Revised implementation → resubmit
    → Approved → merge → monitor convergence accuracy
    → Outcome: convergence accuracy improved 12%
    → MoodSignal: pleasure +0.3, dominance +0.2 (satisfaction + confidence)
    → MindMap: "adaptive threshold technique: validated, 12% improvement"
    → Strategy learning: "convergence improvements benefit from paper research first"
```

### 6.3 Cross-Session Memory Continuity

```
Session 1:
    Agent discovers paper on adaptive signal decay
    → MindMap node: "adaptive signal decay paper" (arousal: +0.4, excitement)
    → Attempts implementation, tests fail
    → MindMap edge: paper → ConvergenceDetector (APPLIES_TO, failed)
    → MoodSignal: pleasure -0.2 (disappointment)
    → Session ends

    [Consolidation runs between sessions]
    → ExperienceConsolidationPhase promotes key events
    → Curiosity signal generated: "adaptive decay — referenced but incomplete"
    → Reflection: "I tried adaptive decay but it failed. The half-life was too aggressive."

Session 2 (weeks later):
    Agent starts. Loads narrative, MindMap, mood state.
    → Inner narrative: "I previously tried adaptive decay — it failed because the half-life was too aggressive"
    → Curiosity signal active: "adaptive decay — referenced but incomplete"
    → CURIOSITY drive elevated (0.6)
    → Searches for updated research: finds paper suggesting per-signal adaptive rates
    → MindMap: new paper node linked to previous failure (BUILDS_ON edge)
    → Emotional association: cautious optimism (arousal: +0.2, pleasure: +0.1)
    → Implements per-signal adaptive rates with conservative defaults
    → Tests pass → PR submitted → approved → merged
    → MindMap: technique validated (pleasure: +0.3)
    → Previous failure node updated: "resolved by per-signal approach"
    → Reflection: "persistence paid off — the second paper addressed the exact gap"
```

---

## 7. Feasibility Analysis

### 7.1 What's Proven (Shipping Code)

| Component | Status | Evidence |
|-----------|--------|----------|
| Case lifecycle (bindings, workers, SubCase) | Production | 200+ tests, multiple execution models |
| Signal/pheromone model with decay and consensus | Production | SignalRegistry, 50+ tests |
| Self-provisioning with budget enforcement | Production | SwarmProvisioner, D84-D91 |
| CognitionCore with tick-based lifecycle | Production | 8 subsystems, tested |
| Drive system (4 axes, modulation, composition) | Production | DriveOrchestrator, DriveComposer |
| PAD mood model with decay and appraisal | Production | MoodOrchestrator, MoodState |
| MindMap with PAD on every node | Production | SQLite + in-memory backends |
| Consolidation with corroboration gating | Production | ExperienceConsolidationPhase |
| Curiosity signal generation | Production | CuriositySignalGenerator, 6 categories |
| Goal formation from drives | Production | GoalProposalOrchestrator, DriveGoalMapper |
| DevTown code review | Production | Operational with API |
| Goal formation, revision, abandonment | Production | GoalFormationEvaluator, GoalRevisionEvaluator |

### 7.2 What's Novel (Integration)

| Integration | Risk | Mitigation |
|-------------|------|------------|
| Improvement-specific DriveSource implementations | Low | Same SPI as existing drives; different input signals |
| ImprovementBudget enforcement | Low | Same pattern as ProvisionBudget |
| Case-as-improvement with conditional bindings | Low | SubCaseBinding and binding triggers are proven |
| Rule-based workers (deps, lint, coverage, CI, recipes) | Medium | API-mediated; each testable independently |
| CognitionCore pointed at self-improvement | Medium | Integration, not invention — but the emergent behaviour is unvalidated |
| Research loop (search → analyse → plan) | Medium-High | Search APIs are mechanical; synthesis requires LLM; feasibility depends on LLM quality |
| MindMap as complete lived experience for self-improvement | Medium | MindMap exists; mapping all improvement experiences into it is new |
| Mood-driven prioritisation vs rule-based allocation | Unknown | The key question: does cognitive depth produce better improvement decisions? Testable: A/B comparison |

### 7.3 The Core Feasibility Question

**Does cognitive depth produce better self-improvement decisions than mechanical rule-following?**

The honest answer: we don't know yet. The research suggests yes:
- SELAgents [9] showed 49% improvement in emotional intelligence, 66% in social coherence
- Park et al. [21] showed reflection was critical for long-horizon behavioural coherence
- The "Drop the Hierarchy and Roles" paper [17] showed role autonomy outperforms rigid structure by 14%

But none of these studied cognitive architecture applied specifically to self-improvement. This is genuinely novel territory.

### 7.4 The Design Principle: Head in the Clouds, Feet on the Ground

The mitigation for all integration risk is a single design principle: **every cognitive capability is independently configurable, independently measurable, and independently toggleable.**

`CognitionConfig` already provides this pattern — `CognitionConfig.all()` for full depth, `.none()` for mechanical mode, `.without(narrative, mentalModel)` for anything in between. The self-improvement system follows the same pattern:

| Layer | What it adds | Toggle | Measure |
|-------|-------------|--------|---------|
| Engine foundation | Case lifecycle, rule-based workers, DevTown, CBR | Always on | Merge rate, regression rate |
| Drive-based prioritisation | Balanced competing needs replace fixed rules | `drivesEnabled` | Does it pick better targets? |
| PAD mood feedback | Emotional response to outcomes modulates drives | `moodEnabled` | Does emotional modulation reduce regressions? |
| Narrative continuity | Inner monologue for cross-session coherence | `narrativeEnabled` | Does temporal coherence improve decision quality? |
| Strategy learning | Learn what improvement approaches work | `strategyEnabled` | Does the approval rate improve over time? |
| Mental model | Track reviewer preferences, platform fragility | `mentalModelEnabled` | Does it avoid known-fragile areas? |
| Research loop | External knowledge acquisition | `curiosityEnabled` + research bindings | Do research-informed improvements outperform rule-based? |
| Full cognitive agent | Everything on, inner life enabled | `CognitionConfig.all()` | Full A/B comparison vs mechanical baseline |

Each layer is independently toggleable. If 6 coupled feedback loops produce unpredictable behaviour — reduce to 2. If mood-congruent retrieval causes negative spirals — disable mood modulation on retrieval. If the research loop doesn't produce actionable improvements — turn it off.

This resolves the adversarial reviewer's core concerns:
- **Emergent behaviour unpredictability (#4):** Toggle off the layers producing unpredictable interactions
- **Testing problem (#5):** Each layer has its own A/B metric; composition is tested by progressively enabling layers
- **Grounding problem (#6):** Domain transfer from social cognition is validated per-layer, not all-at-once
- **Scope and delivery risk (#7):** Each layer is independently deliverable and independently valuable

The platform aspires to the full cognitive stack — head in the clouds. But it can be configured for feet on the ground at any moment, at any granularity.

### 7.5 Open Risk: Approval Optimisation (Goodhart's Law)

One adversarial concern is NOT resolved by configurability: the system may optimise for DevTown approval rather than actual improvement quality. Successful PRs produce positive pleasure, reinforcing patterns that led to approval. Over time, the system learns "small, safe, incremental changes get approved; large systemic changes get rejected" and shifts toward the former. The platform improves at the margins but never tackles systemic issues.

This is Goodhart's Law applied to self-improvement: when approval becomes the measure, the system optimises for approval rather than for quality. Mitigations to explore:

1. **Improvement impact metrics** alongside approval rate — track not just "was the PR approved?" but "did coverage/stability/performance actually improve after merge?"
2. **Diversity incentives** — the ImprovementBudget could reserve capacity for high-risk/high-reward improvements, preventing convergence on safe-only changes
3. **Periodic human review of improvement patterns** — a meta-review that asks "is the system tackling the right problems, or gaming approval?"
4. **Goal revision from outcome metrics** — GoalRevisionEvaluator adjusts improvement direction based on platform health trends, not just PR outcomes

This risk is structural and deserves dedicated attention in the spec. It is flagged here as an open design problem.

### 7.6 The Human Interaction Thesis

There is a second value thesis independent of automated decision quality: **cognitive agents are better collaboration partners for humans.**

Even if automated decisions are no better than simple rules, the cognitive architecture transforms how humans interact with the self-improvement system. A mechanical system reports status: "CI red, 3 tests failing, proposing dependency bump." A cognitive agent shares experience: "I've been struggling with the planning module — three PRs rejected this month. But I found a paper on adaptive decay that connects to something I tried before. I'm cautiously optimistic about the per-signal variant."

That conversation surfaces things a dashboard never could:

- **Experience history the human didn't know about.** "I tried this before — it failed because the half-life was too aggressive." The human learns what the agent has explored, what worked, what didn't — without reading logs.
- **Emotional associations that reveal judgment.** "I've been avoiding the routing module." Is that avoidance justified, or is the agent being irrationally cautious? The human can probe and redirect.
- **Research connections the human wouldn't have made.** "This new paper connects to my past failure." The agent's associative memory surfaces cross-temporal insights that linear reporting misses.
- **Shared direction-finding.** "I'm cautiously optimistic." The human can validate or challenge that assessment. The conversation produces directions neither human nor agent would have found alone.

The cognitive state creates a shared mental model between human and agent. The agent's narrative, emotions, and memory give the human something to react to, challenge, redirect, and build on. This is qualitatively different from interacting with a tool — it's collaborating with a partner that has its own perspective.

This value scales with model capability just as automated decisions do. A more capable model produces richer self-reflection, more nuanced emotional expression, and more insightful research connections — all of which make the human-agent conversation more productive. The cognitive architecture is a capability amplifier for collaboration, not just for automation.

The "anthropomorphic theatre" critique may be technically correct — the system doesn't feel anything, it adjusts floating-point values. But if those floating-point values, rendered through prompt sections and interpreted by a capable LLM, produce conversations that lead to better engineering outcomes through human-agent collaboration, then the theatre is doing real work. The question is not "does the system genuinely feel frustration?" but "does the human-agent conversation that results from modelling frustration produce better directions than a status dashboard?"

---

## 8. Implementation Roadmap

### Epic Decomposition

| Epic | Scope | Repo | Standalone value | Depends on |
|------|-------|------|-----------------|------------|
| 1. Engine foundation | Case lifecycle, signals, ImprovementBudget, rule-based workers, DevTown capability | engine | Full operational self-improvement | — |
| 2. Cognitive agent | CognitionCore integration, improvement DriveSource implementations, PAD feedback from outcomes | blocks + engine | Emotionally and motivationally informed decisions | Epic 1 |
| 3. Research loop | Search infrastructure, paper retrieval, LLM synthesis, research MindMap nodes | blocks + neocortex | External knowledge acquisition | Epic 1 |
| 4. Full cognitive memory | Complete MindMap integration (builds, interactions, relationships), consolidation, emotional associations | neocortex | Cross-session wisdom | Epic 2 |
| 5. Continuous evolution (#1115) | Standing directive, feedback loop, growth direction, data autophagy prevention | engine + blocks | Autonomous operation | Epics 1-4 |

Each epic is independently valuable. Epic 1 delivers a working self-improvement system. Epics 2–4 make it progressively smarter. Epic 5 makes it autonomous.

---

## 9. Conclusion

The cognitive self-improvement architecture is not a single feature — it's a thesis: that self-improving systems benefit from cognitive depth, and that the cognitive infrastructure CaseHub already ships provides that depth. The thesis is testable (does cognitive prioritisation outperform mechanical allocation?), the infrastructure is proven (every component exists as shipping code), and the risk is bounded (the engine foundation works regardless of whether cognitive integration helps).

What makes this potentially extraordinary is the composition. No individual component is novel — drive systems, PAD emotions, MindMap graphs, consolidation, goal formation all exist independently. What's novel is pointing all of them at self-improvement and asking: what happens when an agent that thinks, feels, remembers, and narrates is tasked with improving itself?

The answer is either "nothing useful — stick to the rules-based system" or "something fundamentally different from what any self-improving agent has achieved before." Both outcomes advance understanding. But only one of them changes what self-improving systems can do.

---

## References

[1] Zhang, J.Z. et al. "Darwin Gödel Machine: Open-Ended Evolution of Self-Improving Agents." arXiv:2505.22954, May 2025. https://arxiv.org/abs/2505.22954

[2] Robeyns, M., Szummer, M., Aitchison, L. "A Self-Improving Coding Agent." ICLR 2025 Workshop (SSI-FM). arXiv:2504.15228. https://arxiv.org/abs/2504.15228

[3] Novikov, A. et al. "AlphaEvolve: A coding agent for scientific and algorithmic discovery." arXiv:2506.13131, June 2025. https://arxiv.org/abs/2506.13131

[4] Rahmani et al. "Architectural Patterns for Self-Improving Intelligent Agents: A Systematic Survey of Generative and Cognitive Approaches." SN Computer Science, 2026. https://link.springer.com/article/10.1007/s42979-026-05333-6

[5] Fang, J. et al. "A Comprehensive Survey of Self-Evolving AI Agents." arXiv:2508.07407, August 2025. https://arxiv.org/abs/2508.07407

[6] "Self-Improvements in Modern Agentic Systems." arXiv:2607.13104, July 2026. https://selfimproving-agent.github.io/

[7] "Recursive Self-Improvement in AI: From Bounded Self-Refinement to Autonomous Research Loops." arXiv:2607.07663, July 2026. https://arxiv.org/html/2607.07663v1

[8] Mehrabian, A. & Russell, J.A. "An approach to environmental psychology." MIT Press, 1974.

[9] "Social and emotional learning in artificial agents." Nature Scientific Reports, April 2026. https://www.nature.com/articles/s41598-026-48309-5

[10] "Sentipolis: Emotion-Aware Agents for Social Simulations." arXiv:2601.18027, January 2026. https://arxiv.org/html/2601.18027v1

[11] "A generic self-learning emotional framework for machines." Scientific Reports, 2024. https://www.nature.com/articles/s41598-024-72817-x

[12] Montag et al. "On the relevance of Maslow's need theory in the age of artificial intelligence." ScienceDirect, 2025. https://www.sciencedirect.com/science/article/pii/S0040162525002537

[13] Ogot, M. "A Maslow-Inspired Hierarchy of Engagement with AI Model." arXiv:2509.07032, 2025. https://arxiv.org/pdf/2509.07032

[14] "The Orchestration of Multi-Agent Systems: Architectures, Protocols, and Enterprise Adoption." arXiv:2601.13671, January 2026. https://arxiv.org/html/2601.13671v1

[15] "Swarm Intelligence for AI Agents: Coordination Patterns, Failure Modes, and Production Reality." Zylos Research, May 2026. https://zylos.ai/research/2026-05-23-swarm-intelligence-multi-agent-coordination-patterns/

[16] "A collective intelligence model for swarm robotics applications." Nature Communications, 2025. https://www.nature.com/articles/s41467-025-61985-7

[17] "Drop the Hierarchy and Roles: How Self-Organizing LLM Agents Outperform Designed Structures." arXiv:2603.28990, March 2026. https://arxiv.org/abs/2603.28990

[18] "AI Agent Memory Architectures: From Context Windows to Persistent Knowledge." Zylos Research, April 2026. https://zylos.ai/research/2026-04-05-ai-agent-memory-architectures-persistent-knowledge/

[19] "State of AI Agent Memory 2026: Benchmarks & Trends Report." Mem0, 2026. https://mem0.ai/blog/state-of-ai-agent-memory-2026

[20] "Graph-based Agent Memory: Taxonomy, Techniques, and Applications." arXiv:2602.05665, February 2026. https://arxiv.org/html/2602.05665v1

[21] Park, J.S. et al. "Generative Agents: Interactive Simulacra of Human Behavior." UIST '23, ACM, 2023. https://dl.acm.org/doi/fullHtml/10.1145/3586183.3606763

[22] "Synthetic emotions and consciousness: exploring architectural boundaries." AI & Society, 2026. https://link.springer.com/article/10.1007/s00146-026-02896-z

[23] "Emotion-Integrated Cognitive Architectures: A Bio-Inspired Approach to Developing Emotionally Intelligent AI Agents." ResearchGate, 2024. https://www.researchgate.net/publication/378189406

[24] Gao, H. et al. "A Survey of Self-Evolving Agents." TMLR 2026. arXiv:2507.21046. https://arxiv.org/abs/2507.21046
