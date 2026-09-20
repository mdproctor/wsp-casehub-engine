# Cognitive Self-Improvement — Vision Spec

**Epic:** casehubio/engine#1104 (Hive Mind)
**Scope:** Parent vision — governs epics 1–5
**Date:** 2026-09-20
**Decisions:** D92–D105
**Research:** `2026-09-20-cognitive-self-improvement-research.md`
**Adversarial review:** `2026-09-20-adversarial-review.md`

## The Premise

A platform that coordinates agents should be able to improve itself using those same agents. Not as a special case — as a natural consequence of the architecture.

CaseHub already has: cases with binding-driven lifecycles, agents with capabilities and trust, signals for coordination, goals for intention, and a cognitive stack (blocks + neocortex) that gives agents personality, emotion, memory, and learning. Self-improvement wires these together so the platform treats its own evolution as a case to be managed, using the same machinery it uses for everything else.

The system that emerges is not a maintenance bot. It is a cognitive agent that perceives quality gaps, feels urgency about degradation, remembers what worked and what failed, learns from research, proposes improvements through its own goal system, executes them through the case lifecycle, submits them for review, and integrates the outcomes into its evolving understanding of the platform. It is the platform's immune system, researcher, and craftsman — running on infrastructure the platform already provides.

## Design Principles

### Head in the Clouds, Feet on the Ground

Every cognitive capability is independently configurable, independently measurable, and independently toggleable via `CognitionConfig`. The vision is the full cognitive agent. The reality is incremental delivery — each epic adds cognitive depth on top of a working mechanical foundation.

Engine-only mode works: rule-based detection, budget enforcement, case lifecycle, outcome tracking. Blocks adds cognition: mood, drives, narrative, strategy. Neocortex adds memory: persistent knowledge, emotional associations, curiosity-driven exploration. Each layer enhances — none is required for the layer below it to function.

### SPIs, Not Implementations

The engine consumes the cognitive stack through SPIs — `DriveSource`, `DriveGoalMapper`, `DriveGoalFormationStrategy`, `ConsolidationPhase`, `CuriositySignalProvider`, `CognitionTickParticipant`, `ModulationFactor`. These are contracts. Implementations evolve independently across blocks and neocortex.

**This is load-bearing.** The neocortex cognitive capabilities are under active development. During the implementation of this epic sequence, new capabilities will land: goal cognition (#345), enhanced consolidation (#336), cognitive node classification (#322), relationship memory enhancements (#184, #186). The engine must absorb these improvements without spec revision — the SPI boundary is the isolation layer.

When the spec references a cognitive capability, it references the SPI contract. When implementation calls a cognitive service, it calls the SPI. When tests verify cognitive integration, they test through the SPI. The implementation on the other side of that boundary will change. That is by design.

### Capability Evolution Protocol

Because the cognitive stack evolves during implementation:

1. **Before implementing a cognitive integration point:** read the current SPI surface in blocks-core and neocortex. The inventories (`cognitive-stack-inventory.md`, `memory-stack-inventory.md`) capture a snapshot — the code is authoritative.
2. **When a new blocks/neocortex capability lands:** evaluate whether it creates a better integration path for any self-improvement component. If so, update the child spec and implementation — don't wait for a future epic.
3. **Extension points over hardcoded wiring.** Where the spec says "engine provides a rule-based implementation," the implementation should also accept a blocks-provided alternative through the SPI. The rule-based version is the default, not the only option.

### Case-as-Improvement

Each improvement is a case — the same lifecycle, binding triggers, event logging, and routing infrastructure that governs all CaseHub work. An improvement case template defines the steps (introspect, research, implement, review, integrate). Each step is a capability-routed worker. The template is the same hybrid Java/YAML format as every other case definition.

This is not a metaphor. Improvements ARE cases. They appear in case queries, produce EventLog entries, carry audit trails, and obey trust constraints. The entire improvement lifecycle is visible through the same tooling as any other case.

### Signals Sense, Goals Act

Signals are sensors — agents deposit observations about quality, gaps, and opportunities. Signal consensus validates observations (same pattern as `swarm:need-capacity` in #1113). Goals are actuators — once consensus is reached, `GoalFormationService.propose()` creates a `SELF_IMPROVEMENT` goal that enters the standard goal lifecycle.

The separation is architectural: an agent good at noticing a code smell might not be good at fixing it. Signals let any agent contribute observations; goals let the best-suited agent execute. Each layer evolves independently.

## Architecture — The Closed Cognitive Loop

```
Platform state (CI, coverage, lint, deps, research landscape)
    │
    ▼
Sensing (rule-based metrics + swarm agent observations)
    │
    ▼
Signals (improvement:stability:*, improvement:quality:*, improvement:capability:*)
    │
    ▼
Signal consensus (multiple sources reinforce — validated observation)
    │
    ▼
Goal formation (GoalKind.SELF_IMPROVEMENT — budget-gated)
    │
    ▼
Improvement case (spawned via SubCaseBinding)
    │
    ├─→ [operational] introspect → implement → submit-pr → review → integrate
    │
    └─→ [capability]  introspect → research → analyse → implement → submit-pr → review → integrate
            │
            ▼
Outcome (EventLog record → signal projection → CBR trace → MindMap node)
    │
    ├─→ Mood shift (PAD — success = pleasure, failure = frustration)
    ├─→ Drive modulation (mood × personality × narrative → next priorities)
    ├─→ Strategy learning (what approaches work → StrategyProfile)
    ├─→ Memory consolidation (episodic → semantic with emotional valence)
    └─→ Curiosity signals (knowledge gaps → research direction)
            │
            ▼
        Next sensing cycle (shifted by everything above)
```

The loop is closed: outcomes change what the agent perceives, how it feels, what it prioritises, and what it remembers. This is not a scheduled job that runs lint and files PRs. It is a cognitive system whose improvement behaviour emerges from its ongoing experience.

## Improvement Taxonomy

Two dimensions, each with distinct detection and execution characteristics:

### Operational Improvements

Target code hygiene and infrastructure. Detected via rule-based metrics. Executed by engine rule-based workers via REST/GraphQL/MCP APIs.

| Category | Signal source | Worker | API surface |
|----------|--------------|--------|-------------|
| Dependency updates | staleness metrics, security advisories | `DependencyUpdateWorker` | Package manager APIs |
| Lint/checkstyle fixes | violation reports | `LintFixWorker` | Build tool output |
| Coverage gaps | coverage reports | `CoverageGapWorker` | Coverage tool APIs |
| CI triage | build failure logs | `CITriageWorker` | CI platform APIs |
| Code recipes | pattern detection | `RecipeWorker` | AST/OpenRewrite APIs |

### Capability Improvements

Target the swarm's cognitive and execution abilities. Detected via drive intensity (curiosity, competence, autonomy, affiliation). Include a research-driven growth loop:

1. Identify opportunity (internal metrics OR proactive exploration)
2. Research (internet, Google Scholar, arXiv — structured API calls)
3. Analyse (evaluate applicability, synthesise findings — LLM)
4. Plan (design architectural improvement — LLM)
5. Implement → submit-pr → review → integrate (standard lifecycle)

Engine provides search infrastructure (rule-based API calls). Blocks provides understanding (LLM-powered synthesis and design). The case template uses conditional bindings — the research/analyse phase fires for capability improvements (`improvementType == 'capability'`) and skips for operational improvements.

## Safety Model

Layered budget enforcement following the `ProvisionBudget` + `DispatchBudget` pattern from #1113:

| Layer | What it constrains |
|-------|-------------------|
| `ImprovementBudget` | Concurrent improvements, daily cap, cooldown, allowed repos, denied paths, PR size limits |
| Structural self-modification denial | Hardcoded (not configurable): improvement infrastructure cannot modify its own safety constraints |
| DevTown review gate | Code review as a standard capability worker — mandatory before integration |
| Integration hard gate | Defensive pre-flight: `REVIEW_COMPLETED` event required in EventLog before integration worker executes |
| Trust model | Standard agent trust constraints apply to all improvement workers |

The improvement system cannot weaken its own guardrails. `ImprovementBudgetEnforcer` hardcodes denied paths for all safety infrastructure. This is not a policy choice — it is a structural constraint.

## Drive System Integration

Self-improvement prioritisation uses the existing Drive system rather than a rigid hierarchy:

| Drive axis | What feeds it | What it produces |
|------------|--------------|-----------------|
| COMPETENCE | CI status, test pass rate, lint violations, coverage | Stability/quality improvement goals |
| CURIOSITY | Research opportunities, knowledge gaps, success plateaus | Research/exploration goals |
| AUTONOMY | Repeated failure classes, capability gaps, trust plateaus | Capability expansion goals |
| AFFILIATION | Team coherence, coordination failures | Coordination improvement goals |

`DriveOrchestrator.tick()` evaluates all four axes. `DriveComposer` modulates by mood and personality. The dominant drive emerges from context — no axis has hardcoded priority. Budget allocation is proportional to drive intensity.

Engine provides `ImprovementDriveSource` implementations for each axis — rule-based evaluators that feed drive intensity from metrics. These implement `DriveSource` (existing blocks SPI). Blocks can enhance with LLM-powered assessment.

## Outcome Tracking — Three-Layer Event-Sourced Model

| Layer | Consumer | Timescale | Infrastructure |
|-------|----------|-----------|----------------|
| Structured EventLog record | Case lifecycle, goal evaluators, budget tracking | Immediate | `EventLog` (existing) |
| Signal projection | Evaluation cycle — swarm adjusts behaviour | Minutes–hours (signal decay) | `SignalRegistry` (existing) |
| CBR trace | Historical learning — future improvements retrieve past outcomes | Persistent | `CbrCaseMemoryStore` (neocortex) |

All three project from the same improvement outcome event. The EventLog record is the source of truth. The same event-sourcing pattern as `CaseLedgerEventCapture`.

## Multi-Epic Roadmap

Each epic is independently deliverable. Each adds cognitive depth on top of the previous.

### Epic 1: Engine Foundation (#1114)

**Repo:** engine
**Delivers:** The complete single-shot improvement cycle.

- Improvement signal types and signal→goal formation bridge
- `ImprovementBudget` + `ImprovementBudgetEnforcer` (layered, structural deny-list)
- Improvement case template (hybrid Java/YAML, conditional bindings)
- Five rule-based worker categories
- DevTown as standard `code-review` capability worker
- Outcome tracking (all three layers)
- `GoalKind.SELF_IMPROVEMENT`
- SPI extension points for epics 2–5

**End state:** The engine can detect an operational issue (stale dependency, lint violation, coverage gap), form an improvement goal, execute the improvement through a case lifecycle, submit for review, integrate on approval, and record the outcome. All without blocks or neocortex.

### Epic 2: Cognitive Agent (blocks + engine)

**Repo:** blocks, engine
**Delivers:** The self-improvement agent as a full cognitive entity.

- `CognitionCore` integration for the self-improvement agent
- `ImprovementDriveSource` implementations wired to cognitive drive system
- PAD feedback loop — improvement outcomes generate mood signals
- Personality configuration via `AgentDescriptor.disposition()`
- Narrative modulation — persistent improvement themes influence priorities
- Strategy learning — improvement approaches accumulate in `StrategyProfile`

**End state:** The self-improvement agent has personality, emotion, and learning. Its improvement decisions are influenced by how it feels, what it needs, and what approaches have worked. A frustrated agent shifts toward stability. An excited agent invests in research.

### Epic 3: Research Loop (blocks + neocortex)

**Repo:** blocks, neocortex
**Delivers:** Automated research infrastructure.

- Search infrastructure — structured API calls to search engines, paper repositories
- Paper retrieval and LLM-powered synthesis
- Architectural analysis — evaluate applicability to CaseHub
- Research case bindings (conditional on `improvementType == 'capability'`)
- MindMap integration — research findings as semantic knowledge nodes

**End state:** The swarm can identify knowledge gaps, search the literature, evaluate techniques, and propose implementations based on what it discovers. The automated version of how CaseHub is actually built.

### Epic 4: Full Cognitive Memory (neocortex)

**Repo:** neocortex
**Delivers:** MindMap as the agent's lived experience.

- Build events, CI outcomes, PR feedback, coordination events as episodic memories
- Consolidation from ephemeral experience to durable knowledge
- Emotional associations on all memory nodes (PAD from formation context)
- Curiosity signals from knowledge graph gaps → research direction
- Mood-congruent retrieval (current emotional state biases what is recalled)
- Relationship memory — per-agent-pair interaction tracking

**End state:** The agent remembers everything — not just research, but builds, reviews, failures, relationships. Unreinforced memories decay. Successful patterns consolidate into permanent knowledge. The emotional landscape guides what the agent finds worth pursuing.

### Epic 5: Continuous Evolution (#1115)

**Repo:** engine, blocks
**Delivers:** The autonomous loop.

- Standing directive / continuous trigger
- Outcome→detection feedback loop
- Prioritisation across concurrent improvements
- Data autophagy prevention
- Growth direction — the swarm decides what to improve
- Rollback on regression

**End state:** Self-improvement runs continuously. The swarm autonomously detects, researches, implements, reviews, and integrates improvements — adjusting direction based on accumulated experience.

## Capability Surface — What Engine Consumes

These are the SPI contracts that engine depends on. Implementations live in blocks and neocortex. This list is a snapshot — check the current code before implementing each integration point.

### From blocks-core

| SPI | Engine uses it for | Current state (2026-09-20) |
|-----|--------------------|---------------------------|
| `DriveSource` | Feed improvement metrics into drive system | Shipping — 4 built-in implementations |
| `DriveGoalMapper` | Map improvement drive intensity to goals | Shipping — `CuriosityGoalMapper` exists |
| `DriveGoalFormationStrategy` | LLM-backed goal formation from drive context | Shipping |
| `CognitionTickParticipant` | Custom cognition tick logic | Shipping (#161) |
| `CognitionConfig` | Feature-flag cognitive subsystems | Shipping |
| `DriveConfig` | Per-axis weights and modulation strengths | Shipping |
| `PromptSection` | Inject cognitive state into agent prompt | Shipping — 9 concrete sections |

### From neocortex

| SPI | Engine uses it for | Current state (2026-09-20) |
|-----|--------------------|---------------------------|
| `CbrCaseMemoryStore` | Store/retrieve improvement outcome traces | Shipping |
| `CbrCaseRetriever` | Similarity-based retrieval of past outcomes | Shipping — with temporal/scope decay |
| `MindMapStore` | Semantic knowledge graph for improvement experience | Shipping — inmem + sqlite backends |
| `ConsolidationPhase` | Promote ephemeral improvement experience to knowledge | Shipping |
| `CuriositySignalProvider` | Alternative curiosity signal sources | Shipping |
| `ModulationFactor` | Bias retrieval by cognitive state | Shipping — 4 built-in factors |
| `MoodState` | Emotional state record with PAD dimensions | Shipping |
| `ExperienceEvent` | Typed experience records (Observation/Action/Outcome) | Shipping |
| `RelationshipEvent` | Inter-agent interaction tracking | Shipping |
| `ReflectionSynthesizer` | LLM-backed insight extraction | Shipping |

### Active evolution (known in-progress)

| Capability | Issue | Relevance |
|------------|-------|-----------|
| Goal cognition | neocortex#345 | Affective valuation, goal-conditioned retrieval, dependency graphs — enriches goal formation |
| Enhanced consolidation | neocortex#336 | Better episodic→semantic promotion — enriches memory |
| Cognitive node types | neocortex#322 | Typed beliefs, intentions, predictions — enriches mental model |
| Relationship memory | neocortex#184, #186 | Reflective diary, per-agent history — enriches AFFILIATION drive |
| Trust evolution | blocks#65 | Trust config in agentic-yaml — enriches safety model |
| Structured agent invoker | blocks#287 | Typed LLM invocation — enriches LLM-backed workers |
| Keyed tick locking | blocks#288 | Per-key locking utility — enriches concurrent tick safety |

**Protocol:** When any of these land, evaluate whether they improve a self-improvement integration point. If so, use them — don't wait for a future epic to "officially" incorporate them. The SPI boundary exists precisely so that new implementations can slot in without spec changes.

## Open Risks

### Goodhart's Law

The self-improvement system optimises for metrics. If metrics diverge from actual quality, the system optimises for the wrong thing. Mitigations:

- Multiple independent metrics (coverage + lint + CI + dependency freshness) — gaming one doesn't game all
- Human review gate (DevTown) — a human or LLM reviewer catches metric-gaming
- Budget limits prevent runaway optimisation
- Outcome tracking detects regressions — a "successful" improvement that causes downstream failures is recorded as negative

This is the strongest attack angle (identified in adversarial review). It is structurally mitigated but not eliminated. Monitoring the gap between metrics and actual quality is a continuous concern.

### Complexity Budget

Five epics across three repos is substantial scope. Mitigations:

- Each epic is independently deliverable and independently valuable
- Engine-only mode (Epic 1) works without blocks or neocortex
- Each subsequent epic adds cognitive depth, not functional breadth
- The SPI boundary means epics can be developed concurrently

### Anthropomorphic Language

The spec uses emotional and cognitive language ("feels," "remembers," "learns"). This is not poetic license — it describes actual system behaviour. The PAD model produces measurable emotional states. The MindMap stores retrievable memories. The strategy system accumulates verifiable learning. The language maps to code. Where it stops mapping to code, the system stops making claims.
