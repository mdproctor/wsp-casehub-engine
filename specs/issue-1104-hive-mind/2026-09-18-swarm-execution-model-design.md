# Swarm Execution Model — Design Spec

**Issue:** casehubio/engine#1112
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-18
**Decisions:** D73–D83

## Problem

The stigmergy execution model (#1111) provides indirect coordination via shared environment modification — agents observe, deposit signals, register interests, evaluate local rules, and the pipeline drives the perceive→decide→act cycle. But the model treats agents as independent entities with no awareness of their collective behavior:

| Gap | What's missing |
|-----|---------------|
| Role emergence | No tracking of what behavioral patterns agents spontaneously adopt |
| Team detection | No awareness of which agents coordinate more closely with each other |
| Collective progress | No measurement of the swarm's progress toward its mission |
| Agent self-awareness | Agents cannot see system-level metrics (activity rates, budget usage, their own behavioral profile) |

The "Drop the Hierarchy and Roles" paper (arXiv:2603.28990) showed that agents spontaneously created 5,006 unique roles from 8 agents — role emergence is a first-class concern. SwarmSys (arXiv:2510.10047) demonstrated Explorer/Worker/Validator cycles with pheromone-inspired reinforcement. Both require the engine to *observe* emergent patterns, not orchestrate them.

**Scope boundary:** This issue covers tracking and detection only — the engine observes and reports emergent swarm behavior. Dynamic agent scaling is #1113 (self-provisioning). LLM-driven role assignment is blocks issue #10. The engine provides the observation infrastructure; intelligence is layered on top.

## Architecture Overview

The swarm model extends stigmergy (D73) — same `planningStrategy: stigmergy`, no new strategy type. `StigmergyConfig` gains an optional `SwarmConfig swarm` sub-record whose presence activates swarm features. Four new components layer onto the existing stigmergy infrastructure:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Swarm Execution Model                         │
│                                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────────────────┐  │
│  │ RoleTracker  │ │ TeamDetector │ │ SwarmProgressTracker    │  │
│  │              │ │              │ │                         │  │
│  │ • behavioral │ │ • signal-    │ │ • exploration breadth   │  │
│  │   fingerprint│ │   based      │ │ • consensus formation   │  │
│  │ • cosine     │ │   affinity   │ │ • stability score       │  │
│  │   similarity │ │ • emergent   │ │                         │  │
│  │ • role       │ │   clusters   │ │                         │  │
│  │   clusters   │ │              │ │                         │  │
│  └──────┬───────┘ └──────┬───────┘ └────────────┬────────────┘  │
│         │                │                       │               │
│         ▼                ▼                       ▼               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    MetricsSpace                           │   │
│  │   5th WorkerRuntime facet — read-only agent self-awareness│   │
│  │   activityRates · budgetUsage · myFingerprint             │   │
│  │   swarmProgress · detectedRoles                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           Stigmergy Execution Model (#1111)               │   │
│  │  StigmergyConfig · StigmergyStrategy · StigmergyCoord.   │   │
│  │                                                           │   │
│  │  ┌───────────────────────────────────────────────────┐   │   │
│  │  │         Foundation SPIs (#1105-#1110)               │   │   │
│  │  │  SignalRegistry · ObservationRegistry · RuleReg.   │   │   │
│  │  │  InterestSpace · NeighborSpace · ActivityTracker   │   │   │
│  │  │  ConvergenceDetector · BudgetEnforcer              │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

**Why extend, not separate.** D65 explicitly anticipated that `StigmergyStrategy.select()` gains additional logic for #1112 without architectural change. Swarm IS stigmergy plus collective intelligence — role awareness, team detection, progress monitoring. A separate `planningStrategy: swarm` would duplicate stigmergy's dispatch and lifecycle logic. Extending keeps one strategy, one coordinator, one configuration surface.

## 1. RoleTracker

`RoleTracker` (`runtime-core`, `io.casehub.engine.internal.stigmergy`, `@ApplicationScoped`, `Resettable`) — detects emergent roles from agent behavioral patterns.

### First Principles

In biological swarms, roles emerge from four observable dimensions: what the agent responds to (stimulus sensitivity), what the agent does (behavioral patterns), how the agent affects the environment (consequences), and how other agents respond (social feedback). The ant response threshold model shows that roles emerge from *differential sensitivity* — ants with lower thresholds for a stimulus respond first, naturally allocating tasks without assignment.

In CaseHub, the engine already has complete behavioral information across the existing registries. Role detection is a query over existing data, not new data collection.

### Behavioral Fingerprint

`BehavioralFingerprint` (`api`, `io.casehub.api.model.stigmergy`) — structured record with four domain-specific sub-vectors:

```java
record BehavioralFingerprint(
    Map<String, Double> perception,      // interest keys → 1.0 (binary)
    Map<String, Double> communication,   // signal names → normalized deposit count
    Map<String, Double> decision,        // rule IDs → normalized firing count
    Map<String, Double> effect           // context keys → normalized write count
)
```

Each sub-vector is a sparse `Map<String, Double>`. Feature names are plain strings within their domain — no namespace prefixing needed since the domains are separate fields.

| Domain | Features | Source | Type |
|--------|----------|--------|------|
| Perception | interest keys registered | `ObservationRegistry` — extract `watchedKeys()` per observer for this agent | Binary (1.0) |
| Communication | signal names deposited | `SignalRegistry` — signals where `sources` contains this agent | Normalized counts |
| Decision | rule IDs that fired | `RoleTracker` accumulator — sliding window of `RuleFiring` records | Normalized counts |
| Effect | context keys written | `RoleTracker` accumulator — sliding window of `WriteContext` actions | Normalized counts |

**Static features** (perception) change infrequently — agents register interests during `execute()` and rarely modify them. **Dynamic features** (communication, decision, effect) evolve continuously as agents adapt their behavior.

**Normalization:** Dynamic feature values are normalized per-agent: `featureCount / totalAgentActivityInDomain`. An agent that deposits signal "overheating" 80 times out of 100 total deposits has `communication["overheating"] = 0.8`. This makes fingerprints comparable regardless of activity level.

### Similarity Computation

Similarity between two agents' fingerprints is a weighted average of per-domain cosine similarities:

```
similarity = w_p × cos(A.perception, B.perception)
           + w_c × cos(A.communication, B.communication)
           + w_d × cos(A.decision, B.decision)
           + w_e × cos(A.effect, B.effect)
```

Cosine similarity on sparse non-negative vectors: range [0.0, 1.0]. `cos(empty, empty) = 0.0` (no shared features). `cos(A, empty) = 0.0` (one agent has no activity in this domain).

`RoleDomainWeights` record (`api`, `io.casehub.api.model.stigmergy`):

```java
record RoleDomainWeights(
    Double perception,       // default 0.25
    Double communication,    // default 0.25
    Double decision,         // default 0.25
    Double effect            // default 0.25
)
```

Weights are normalized to sum to 1.0 at initialization. Case authors can emphasize specific domains — e.g., `{ communication: 0.5, effect: 0.3, perception: 0.1, decision: 0.1 }` for a case where "what agents communicate" defines roles more than "what they watch."

**Why four domains, not one flat vector.** (1) Different semantics — perception features are binary, communication features are counts. Mixing requires normalization that obscures domain structure. (2) Domain-weighted similarity gives case authors control over what "role" means. (3) Per-domain analysis gives richer audit: "identical perception, divergent effects." (4) Each domain vector is very sparse (≤20 interests, ≤100 signals, ≤50 rules) — per-domain cosine on small sparse vectors is efficient.

### Sliding Window Accumulation

Dynamic features (rule firings, context writes) are ephemeral in the registries — `RuleRegistry` only stores last-cycle firings (per D43). The `RoleTracker` accumulates them in a sliding window.

Per-agent, per-domain: a circular buffer of per-cycle feature maps over the last N evaluation cycles (`roleDetectionWindow`, default 20). On each evaluation cycle: append current-cycle counts, evict oldest cycle if window is full. Normalize by total per-agent activity within the window.

**Storage:** Per-agent circular buffer of per-cycle feature maps. Bounded by `agents × windowSize × featuresPerCycle`. With 20 agents × 20 cycles × 50 features = 20,000 entries. Negligible.

**Accumulation points:**
- Rule firings: recorded by `RuleRegistry` per cycle — `RoleTracker` reads `RuleRegistry.getFirings(caseId, agentId)` each cycle
- Context writes: recorded by `LocalRuleEvaluator` — `RoleTracker` is notified of `WriteContext` actions per agent each cycle

### Role Cluster Detection

Algorithm (runs on periodic + event triggers per D80):

1. Compute `BehavioralFingerprint` for each active agent (from registries + sliding window)
2. Compute pairwise similarity matrix: O(N² × K) where N = agent count, K = avg feature count per domain
3. Build adjacency graph: edge between agents i, j if `similarity(i,j) > roleSimilarityThreshold` (default 0.7)
4. Find connected components via BFS
5. For each component, compute average internal pairwise similarity
6. If average < `roleSimilarityThreshold`, split by removing weakest edge and recurse
7. Components with size ≥ `roleMinClusterSize` (default 2) and avg similarity ≥ threshold are role clusters

With N ≤ 20 agents, total cost is O(400 × K) per detection — negligible.

### DetectedRole

```java
record DetectedRole(
    String roleId,                          // engine-generated, stable across cycles
    Set<String> memberAgents,               // agent IDs in this cluster
    BehavioralFingerprint centroid,         // average fingerprint of cluster members
    List<String> dominantFeatures,          // top-3 features from centroid (for naming)
    int stabilityCount                      // consecutive detection cycles this cluster persisted
)
```

`roleId` is generated on first detection as `"role-" + incrementingCounter` (per case, reset on eviction) and reused across cycles via cluster matching. Two clusters match if they share > 50% of their members (majority overlap).

### Role Evolution Detection

Compare current role cluster set to previous detection cycle's set:

| Event | Condition |
|-------|-----------|
| `SWARM_ROLE_EMERGED` | New cluster detected (no matching cluster in previous set) |
| `SWARM_ROLE_DISSOLVED` | Previous cluster no longer exists (no matching cluster in current set) |
| `SWARM_ROLE_SHIFT` | Agent moved from one cluster to another between detection cycles |

Convergence (two clusters merging) and specialization (one cluster splitting) are derivable from EMERGED/DISSOLVED pairs in the event stream — no dedicated event types.

### Trade-offs

Multi-dimensional fingerprint is more complex than Jaccard or flat vector — four cosine computations per pair instead of one. Mitigated: N ≤ 20 agents, each domain vector is very sparse. The richness of per-domain analysis and configurable domain weights justifies the complexity.

## 2. TeamDetector

`TeamDetector` (`runtime-core`, `io.casehub.engine.internal.stigmergy`, `@ApplicationScoped`, `Resettable`) — detects emergent coordination clusters from inter-agent relations.

### Team vs. Role

Role clusters and team clusters are related but distinct:
- **Role clusters:** agents with similar *behavior* (doing the same thing)
- **Team clusters:** agents with high *interaction* (working together)

Two agents with the same role might not be teaming (parallel independent work). Two agents with different roles might be teaming (complementary collaboration).

### Affinity Computation

Team affinity between two agents is computed from three NeighborSpace relation types:

| Relation | What it measures | Source |
|----------|-----------------|--------|
| `SHARED_INTEREST` | Attention alignment — watching the same context keys | `ObservationRegistry` — compare `watchedKeys()` per observer |
| `SHARED_SIGNAL` | Communication alignment — depositing the same signals | `SignalRegistry` — compare `sources` sets across signals |
| `COMPLEMENTARY` | Workflow alignment — one agent's outputs feed another's observations | Cross-reference output keys (from `WriteContext` actions) against interest keys |

Affinity score: for each relation type, compute a Jaccard-like ratio (`shared features / union features`). Overall affinity is the average of the three scores.

```
affinity(A, B) = (sharedInterestScore + sharedSignalScore + complementaryScore) / 3.0
```

Each component score is [0.0, 1.0]. `sharedInterestScore = |A.interestKeys ∩ B.interestKeys| / |A.interestKeys ∪ B.interestKeys|`. Signal and complementary scores follow the same pattern.

### Team Cluster Detection

Same algorithm as role clustering (connected components + internal average check) but on the affinity matrix instead of behavioral similarity:

1. Compute pairwise affinity matrix
2. Build adjacency graph: edge if `affinity > teamAffinityThreshold` (default 0.5)
3. Connected components with internal average check
4. Components ≥ `teamMinSize` (default 2) are team clusters

### DetectedTeam

```java
record DetectedTeam(
    String teamId,                          // engine-generated, stable across cycles
    Set<String> memberAgents,               // agent IDs in this cluster
    Set<String> dominantRelations,          // which relation types dominate the affinity
    double avgAffinity,                     // average internal affinity score
    int stabilityCount                      // consecutive detection cycles
)
```

Team matching across cycles uses the same majority-overlap rule as roles.

### Team Evolution Events

| Event | Condition |
|-------|-----------|
| `SWARM_TEAM_FORMED` | New team cluster detected |
| `SWARM_TEAM_DISSOLVED` | Previous team cluster no longer exists |

### Pipeline Integration

Runs alongside `RoleTracker` on the same trigger schedule (periodic + event-triggered per D80). Both share the same dirty flag — a structural change (departure, registration) triggers both role and team re-detection.

### Trade-offs

No explicit team identity — agents can't say "I'm in team X." Agents detect team membership indirectly via `NeighborSpace` queries. Acceptable for v1 — explicit team identity is a natural extension if needed.

## 3. SwarmProgressTracker

`SwarmProgressTracker` (`runtime-core`, `io.casehub.engine.internal.stigmergy`, `@ApplicationScoped`, `Resettable`) — tracks collective progress toward the swarm's mission.

### Progress Dimensions

Three generic progress metrics, each [0.0, 1.0]:

| Dimension | What it measures | Computation |
|-----------|-----------------|-------------|
| Exploration breadth | How much of the problem space agents have covered | Ratio of unique features explored (unique signal names + unique context keys written) vs. a sliding maximum. Higher = more exploration. |
| Consensus formation | How much agreement has formed among agents | Ratio of signals with consensus (`sources.size() ≥ consensusThreshold`) vs. total active signals. Higher = more agreement. |
| Stability score | How settled the swarm's role structure is | Percentage of agents that stayed in the same role cluster across the last N detection cycles. Higher = roles have stabilized. |

### SwarmProgress

```java
record SwarmProgress(
    double explorationScore,        // [0.0, 1.0]
    double consensusScore,          // [0.0, 1.0]
    double stabilityScore,          // [0.0, 1.0]
    Instant computedAt
)
```

### Computation

**Exploration breadth:**
- Numerator: count of unique signal names deposited + unique context keys written by all agents in the current sliding window
- Denominator: sliding maximum of this count over the last `roleDetectionWindow` detection cycles (default 20) — avoids division by a static constant that doesn't adapt to case complexity
- At detection cycle 1: score = 1.0 (numerator = denominator). Score decreases if agents stop exploring new features.

**Consensus formation:**
- Numerator: count of signals in `SignalRegistry.perceiveAll(caseId)` where `sources.size() ≥ consensusThreshold`
- Denominator: total count of signals above effective-zero threshold
- Empty registry: score = 0.0

**Stability score:**
- Compare current role cluster membership to previous N detection cycles
- For each agent: 1 if in the same cluster (majority-overlap match) as previous cycle, 0 otherwise
- Score = sum / activeAgentCount
- First detection cycle: score = 0.0 (no history to compare)

### Event

`SWARM_PROGRESS` event fired when any score changes by more than `progressChangeThreshold` (default 0.1) since the last event. Metadata: `explorationScore`, `consensusScore`, `stabilityScore`, `cycle`.

Runs on periodic interval only — progress is a slow-moving metric, not sensitive to structural changes.

### Trade-offs

Generic metrics are proxies, not direct progress measures. A high exploration score doesn't mean the *right* things were explored. Domain-specific progress belongs in goal conditions (`completion: { success: { anyOf: [...] } }`), not the generic tracker. The tracker gives agents enough information via `MetricsSpace` to adapt their exploration/exploitation balance.

## 4. MetricsSpace

`MetricsSpace` (`api`, `io.casehub.api.engine`) — the 5th `WorkerRuntime` facet, providing read-only agent self-awareness.

### Interface

```java
interface MetricsSpace {

    Map<String, Double> activityRates();

    Map<String, Long> budgetUsage();

    BehavioralFingerprint myFingerprint();

    SwarmProgress swarmProgress();

    List<DetectedRole> detectedRoles();

    MetricsSpace NOOP = new MetricsSpace() { /* all methods return empty/default */ };
}
```

| Method | Returns | Source |
|--------|---------|--------|
| `activityRates()` | Current rates for dispatch, signal deposit, context mutation, evaluation | `ActivityTracker` |
| `budgetUsage()` | Cumulative counts for each budget metric | `ActivityTracker` |
| `myFingerprint()` | This agent's current behavioral fingerprint | `RoleTracker` |
| `swarmProgress()` | Swarm-level progress scores | `SwarmProgressTracker` |
| `detectedRoles()` | All currently detected role clusters | `RoleTracker` |

All methods return immutable snapshots. No mutations.

### WorkerRuntime Integration

```java
// WorkerRuntime (api/engine)
default MetricsSpace metrics() {
    return MetricsSpace.NOOP;
}
```

`DefaultMetricsSpace` (`runtime-core`, `io.casehub.engine.internal.stigmergy`) delegates to `ActivityTracker`, `RoleTracker`, `SwarmProgressTracker`. Wired via `WorkerRuntimeFactory` — the factory passes the trackers; the runtime creates the facet.

### D56 Fulfillment

D56 (stigmergy spec) explicitly deferred `MetricsSpace` for #1112: "When swarm scenarios (#1112/#1113) demonstrate a concrete need for agent-level metric visibility, a read-only `MetricsSpace` facet with `activityRates() → Map<String, Double>` is the planned extension path." Swarm IS the concrete use case — agents that can see system-level state make better adaptation decisions:

- See their own fingerprint → compare with role clusters → decide to specialize or diversify
- See swarm progress → decide explore vs. exploit
- See budget usage → voluntarily throttle
- See activity rates → detect system stress

### Trade-offs

Agents can game metrics (e.g., artificially inflate exploration by depositing diverse signals). Mitigated: budget enforcement caps total activity regardless of diversity. The engine is the authority on convergence and budgets, not the agents.

## 5. SwarmConfig

`SwarmConfig` (`api`, `io.casehub.api.model.stigmergy`) — configuration for swarm features, nested inside `StigmergyConfig`.

### Structure

```java
record SwarmConfig(
    Double roleSimilarityThreshold,     // default 0.7
    Integer roleMinClusterSize,         // default 2
    Integer roleDetectionWindow,        // default 20 (sliding window cycles)
    Integer detectionInterval,          // default 10 (periodic detection every N cycles)
    RoleDomainWeights domainWeights,    // default equal (0.25 each)
    Double teamAffinityThreshold,       // default 0.5
    Integer teamMinSize,                // default 2
    Double progressChangeThreshold      // default 0.1 (fire event on score delta)
)
```

All fields nullable — null means use default. Empty `swarm: {}` activates swarm with all defaults.

### StigmergyConfig Integration

`StigmergyConfig` gains a new field:

```java
record StigmergyConfig(
    StigmergyDefaults defaults,
    CoordinationConfig coordination,
    SwarmConfig swarm                    // nullable — presence activates swarm
)
```

Presence of `swarm` activates swarm features (RoleTracker, TeamDetector, SwarmProgressTracker, MetricsSpace). Absence means pure stigmergy without swarm tracking. This follows the presence-as-activation pattern established by `StigmergyConfig` itself (D62).

### YAML

```yaml
stigmergyConfig:
  defaults:
    signalHalfLife: PT5M
    stabilityWindow: PT30S
  coordination:
    consensusThreshold: 2
  swarm:
    roleSimilarityThreshold: 0.7
    detectionInterval: 10
    domainWeights:
      perception: 0.1
      communication: 0.4
      decision: 0.2
      effect: 0.3
```

### Trade-offs

`StigmergyConfig` grows wider (one new nullable field). Mitigated: `SwarmConfig` is a sub-record — pure stigmergy cases don't see it. Swarm without stigmergy is architecturally impossible, so nesting makes the dependency explicit.

## 6. Detection Frequency

Role and team detection use a dual-trigger model (D80):

### Periodic Trigger

Re-cluster every `detectionInterval` evaluation cycles (default 10). Catches gradual behavioral drift in dynamic features (rule firing patterns, signal deposit frequency).

`RoleTracker` maintains a per-case cycle counter. On each `convergenceDetection()` invocation: increment counter, run detection if `counter % detectionInterval == 0`.

### Event Trigger

Immediate re-cluster on structural changes:
- Agent departure (`STIGMERGY_AGENT_DEPARTED`)
- New interest/rule registration
- Interest/rule deregistration

These changes invalidate current clusters immediately. `RoleTracker` maintains a per-case `dirty` boolean flag. Structural changes set the flag. At the next `convergenceDetection()` phase: if dirty, run detection regardless of the periodic counter, then clear the flag.

Detection still runs within the pipeline phase, not inline with the registration event — the dirty flag bridges the event to the next evaluation cycle.

### Separation of Concerns

Fingerprint data accumulates every cycle (cheap counter increments in the sliding window). Clustering runs only on triggers (periodic or event). `SwarmProgressTracker` runs only on periodic intervals — progress is a slow metric not sensitive to individual structural changes.

## 7. Audit Events

Six new `CaseHubEventType` values in three groups:

### Role Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `SWARM_ROLE_EMERGED` | New role cluster detected | `roleId`, `memberAgents`, `dominantFeatures` (top-3), `clusterSize`, `avgSimilarity` |
| `SWARM_ROLE_DISSOLVED` | Role cluster no longer meets minimum membership | `roleId`, `previousMembers`, `lifetimeCycles` |
| `SWARM_ROLE_SHIFT` | Agent moved from one role cluster to another | `agentId`, `fromRoleId`, `toRoleId`, `similarityToNewRole` |

### Team Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `SWARM_TEAM_FORMED` | Team affinity cluster detected | `teamId`, `memberAgents`, `dominantRelations`, `avgAffinity` |
| `SWARM_TEAM_DISSOLVED` | Team cluster no longer exists | `teamId`, `previousMembers`, `lifetimeCycles` |

### Progress Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `SWARM_PROGRESS` | Score delta exceeds threshold or periodic report | `explorationScore`, `consensusScore`, `stabilityScore`, `cycle` |

### The Swarm Intelligence Story

Combined with the stigmergy events (D68), these events tell the complete coordination narrative:

```
t=0:   STIGMERGY_CASE_INITIALIZED       (5 agents, swarm enabled)
t=1:   STIGMERGY_AGENT_JOINED           (temp-monitor, pressure-monitor, ...)
t=3:   STIGMERGY_AGENT_ACTIVATED        (all 5 agents active)
...
t=20:  SIGNAL_CONSENSUS_DETECTED        (overheating, 2 agents)
t=30:  SWARM_ROLE_EMERGED               (role-1: [temp-monitor, pressure-monitor],
                                          dominant: signal:overheating, interest:tempReading)
t=30:  SWARM_ROLE_EMERGED               (role-2: [cooling-controller, vent-controller],
                                          dominant: effect:coolingAction, signal:cooldown)
t=30:  SWARM_TEAM_FORMED                (team-1: [temp-monitor, cooling-controller],
                                          dominant: COMPLEMENTARY)
t=50:  SWARM_PROGRESS                   (exploration=0.8, consensus=0.4, stability=0.6)
t=70:  SWARM_ROLE_SHIFT                 (stability-assessor: ungrouped → role-1)
t=80:  SWARM_PROGRESS                   (exploration=0.9, consensus=0.7, stability=0.9)
t=100: CONVERGENCE_DETECTED             (all rates below threshold for 30s)
```

The audit trail shows role emergence (monitors clustered, controllers clustered), team formation (complementary pairing), role convergence (assessor joining the monitor role), and progress toward completion.

## 8. Module Placement

Extend existing stigmergy packages — no new packages (D82):

| Component | Module | Package |
|-----------|--------|---------|
| `SwarmConfig`, `RoleDomainWeights` | api | `io.casehub.api.model.stigmergy` |
| `BehavioralFingerprint`, `DetectedRole`, `DetectedTeam` | api | `io.casehub.api.model.stigmergy` |
| `SwarmProgress` | api | `io.casehub.api.model.stigmergy` |
| `MetricsSpace` | api | `io.casehub.api.engine` |
| New `CaseHubEventType` values | api | `io.casehub.api.event` (existing enum) |
| `RoleTracker` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `TeamDetector` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `SwarmProgressTracker` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `DefaultMetricsSpace` | runtime-core | `io.casehub.engine.internal.stigmergy` |

Follows the established tier model: API types in Tier 1, infrastructure in Tier 2/3. `MetricsSpace` follows the facet pattern in `api/engine` alongside `SignalSpace`, `InterestSpace`, `NeighborSpace`, `RuleSpace`.

## 9. Work Redistribution

No engine-orchestrated redistribution (D76). Self-healing through the existing perceive→decide→act cycle:

1. Agent fails or departs → `STIGMERGY_AGENT_DEPARTED` event published (D68)
2. Neighboring agents detect departure via local rules — rule condition checks neighbor state via `NeighborSpace` queries in lambda predicates
3. Agents adapt — register additional interests, deposit compensating signals, pick up the departed agent's work
4. `MetricsSpace` enables self-awareness — agents see the departure reflected in detected roles and swarm progress

This is how biological swarms handle worker loss: neighboring ants detect the gap in pheromone refreshment and compensate by adjusting their response thresholds. The engine provides the observation and notification infrastructure; agent rules provide the adaptive logic.

### Trade-offs

Self-healing quality depends entirely on agent rule quality. Poorly-written rules won't compensate for departed agents. Acceptable — the engine provides the infrastructure, blocks (#1112 blocks issue #10) provides the LLM intelligence for sophisticated adaptation.

## 10. Pipeline Integration

The `convergenceDetection()` phase in `CaseContextChangedEventHandler` already runs `StigmergyCoordinator.detectPatterns()` for stigmergy cases. Swarm detection layers on top:

```
convergenceDetection(caseInstance, caseDefinition):
    // Existing stigmergy coordination detection
    if coordinator.isStigmergyCase(caseInstance.id()):
        coordinator.detectPatterns(caseInstance.id(),
            caseDefinition.stigmergyConfig())

    // Swarm detection — guarded by swarmConfig presence
    swarmConfig = caseDefinition.stigmergyConfig()?.swarm()
    if swarmConfig != null:
        roleTracker.accumulate(caseId)           // always — cheap per-cycle accumulation
        if roleTracker.shouldDetect(caseId):      // periodic or dirty flag
            roleTracker.detect(caseId, swarmConfig)
            teamDetector.detect(caseId, swarmConfig)
        if roleTracker.shouldTrackProgress(caseId):  // periodic only
            progressTracker.evaluate(caseId, swarmConfig)

    // Existing budget/convergence detection
    if budgetConfig == null && convergenceConfig == null:
        return
    budgetEnforcer.check(...)
    convergenceDetector.evaluate(...)
```

`RoleTracker`, `TeamDetector`, and `SwarmProgressTracker` are injected into `CaseContextChangedEventHandler` via `Instance<>` with `isResolvable()` guards — transparent no-op when not present. Follows the `Instance<StigmergyCoordinator>` pattern from D70.

### Dependency Injection

All three trackers are `@ApplicationScoped` beans in `runtime-core`. Constructor-injected dependencies:

- `RoleTracker`: `ObservationRegistry`, `SignalRegistry`, `RuleRegistry`, `StigmergyCoordinator`, `EventDispatcher`
- `TeamDetector`: `ObservationRegistry`, `SignalRegistry`, `RoleTracker` (for `WriteContext` action tracking), `StigmergyCoordinator`, `EventDispatcher`
- `SwarmProgressTracker`: `SignalRegistry`, `RoleTracker`, `StigmergyCoordinator`, `EventDispatcher`
- `DefaultMetricsSpace`: `ActivityTracker`, `RoleTracker`, `SwarmProgressTracker`

`DefaultMetricsSpace` is wired via `WorkerRuntimeFactory` — the factory passes the trackers to each `DefaultWorkerRuntime` instance.

### Lifecycle

`CaseStatusChangedHandler` calls `evictByCase(caseId)` on all three trackers on terminal case status. All three implement `Resettable` for demo/test replay.

## 11. Cross-Cutting Concerns

### 11a. Pipeline Decomposition (D83)

The decision review identified that `CaseContextChangedEventHandler` has grown to 34+ constructor parameters with 5 sequential phases. D83 proposes extracting each phase into a dedicated handler class composed into a `CaseEvaluationPipeline`. This is a related architectural improvement that benefits all pipeline phases, not just swarm — it reduces per-class complexity and improves phase testability. Swarm detection adds 3 more `Instance<>` injections to the handler, making the decomposition more pressing.

### 11b. Crash Recovery (D29)

All swarm state is in-memory only — role clusters, team clusters, fingerprint sliding windows, progress scores. On engine restart, all tracking state is lost. Cases that survive a restart lose swarm intelligence but retain the underlying coordination state (signals, interests, rules) which reconstructs naturally as agents re-execute. This is consistent with all coordination state being in-memory (D29).

### 11c. Virtual Thread Compatibility (D72)

All new tracker internals use `java.util.concurrent.locks.ReentrantLock` instead of `synchronized` — consistent with D72's guidance for virtual thread compatibility. Critical sections are short (map operations on per-case state).

## 12. Complete YAML Example

A chemical reactor control case with swarm features — extending the stigmergy example from #1111:

```yaml
dsl: "1.0.0"
namespace: industrial
name: reactor-control-swarm
version: "1.0.0"

spec:
  planningStrategy: stigmergy

  stigmergyConfig:
    defaults:
      signalHalfLife: PT2M
      stabilityWindow: PT30S
      maxDispatches: 5000
    coordination:
      consensusThreshold: 2
      stormRateMultiplier: 10.0
    swarm:
      roleSimilarityThreshold: 0.7
      roleMinClusterSize: 2
      detectionInterval: 10
      domainWeights:
        perception: 0.15
        communication: 0.35
        decision: 0.15
        effect: 0.35
      teamAffinityThreshold: 0.5
      progressChangeThreshold: 0.1

  capabilities:
    - name: monitorTemperature
      outputProjection: "{ tempReading: .reading, tempStatus: .status }"
    - name: monitorPressure
      outputProjection: "{ pressureReading: .reading, pressureStatus: .status }"
    - name: controlCooling
      outputProjection: "{ coolingAction: .action, coolingResult: .result }"
    - name: controlVenting
      outputProjection: "{ ventAction: .action, ventResult: .result }"
    - name: assessStability
      outputProjection: "{ reactorStatus: .status, stabilityScore: .score }"

  workers:
    - name: temp-monitor
      capabilities: [monitorTemperature]
    - name: pressure-monitor
      capabilities: [monitorPressure]
    - name: cooling-controller
      capabilities: [controlCooling]
    - name: vent-controller
      capabilities: [controlVenting]
    - name: stability-assessor
      capabilities: [assessStability]

  bindings:
    - name: run-temp-monitor
      capability: monitorTemperature
    - name: run-pressure-monitor
      capability: monitorPressure
    - name: run-cooling-controller
      capability: controlCooling
    - name: run-vent-controller
      capability: controlVenting
    - name: run-stability-assessor
      capability: assessStability

  goals:
    - name: reactorStable
      condition: '.reactorStatus == "STABLE" and .stabilityScore > 0.9'
      kind: success

  completion:
    success:
      anyOf: [reactorStable, _converged]
```

The `swarm` block activates role tracking, team detection, and progress monitoring. Domain weights emphasize communication (0.35) and effect (0.35) — what agents signal and what they write to context defines roles more than what they watch or which rules fire.

### Java Worker Example

An agent that uses `MetricsSpace` to adapt its behavior:

```java
@Worker(capability = "monitorTemperature")
public class AdaptiveTemperatureMonitor implements WorkerFunction {
    @Override
    public WorkerResult execute(WorkerScope scope) {
        var runtime = (WorkerRuntime) scope;

        // Register base interests and rules
        runtime.interests().register(
            InterestDeclaration.keyThreshold("tempReading", Operator.GT, 100));

        runtime.rules().register(new LocalRule("signal-overheating",
            new PredicateCondition(ctx ->
                ctx.observations().stream()
                    .anyMatch(o -> "threshold_crossed".equals(o.type()))),
            List.of(new DepositSignal("overheating", 0.8)),
            10));

        // Adaptive rule: check metrics and adjust behavior
        runtime.rules().register(new LocalRule("adapt-to-swarm",
            new PredicateCondition(ctx -> {
                var metrics = runtime.metrics();
                var progress = metrics.swarmProgress();
                // If exploration is low, broaden observation
                return progress.explorationScore() < 0.3;
            }),
            List.of(
                new RegisterInterest(
                    InterestDeclaration.keyThreshold("pressureReading", Operator.GT, 50))
            ),
            5));

        // Self-regulation rule: throttle when approaching budget
        runtime.rules().register(new LocalRule("budget-awareness",
            new PredicateCondition(ctx -> {
                var budget = runtime.metrics().budgetUsage();
                var maxDispatches = 5000L;
                return budget.getOrDefault("dispatches", 0L) > maxDispatches * 0.8;
            }),
            List.of(new DepositSignal("throttle-requested", 0.9)),
            20));

        return WorkerResult.of(Map.of("status", "monitoring"));
    }
}
```

After `execute()` returns, the agent's rules run each evaluation cycle. The "adapt-to-swarm" rule checks `MetricsSpace.swarmProgress()` — if exploration is low, it broadens the agent's observation interests. The "budget-awareness" rule checks budget usage and signals for throttling when approaching the cap.

## 13. Relationship to Existing Infrastructure

| Existing | Relationship |
|----------|-------------|
| **StigmergyCoordinator** | Complementary. Coordinator tracks agent lifecycle and coordination patterns. RoleTracker/TeamDetector track behavioral patterns and affinity. Coordinator provides `activeAgents()` queries consumed by all three trackers. |
| **ObservationRegistry** | Consumed. RoleTracker queries interest keys per agent. TeamDetector queries for shared interests. |
| **SignalRegistry** | Consumed. RoleTracker queries signal sources per agent. TeamDetector queries for shared signals. |
| **RuleRegistry** | Consumed. RoleTracker queries rule firings per agent. |
| **ActivityTracker** | Consumed. DefaultMetricsSpace delegates `activityRates()` and `budgetUsage()` to it. |
| **NeighborSpace** (D32-D34) | Complementary. NeighborSpace provides per-agent relational queries. TeamDetector provides system-level cluster detection over those same relations. |
| **OutputConvergenceMonitor** (D51) | Complementary. Tracks output structural similarity for traditional workers. Swarm tracking covers coordination artifact convergence. |
| **WorkerRuntimeFactory** | Extended. Wires `DefaultMetricsSpace` into `DefaultWorkerRuntime`. |
| **engine#1113** (self-provisioning) | Future extension. RoleTracker provides the behavioral intelligence that #1113's provisioning decisions need — which roles are under-represented, which are over-staffed. |
| **blocks#10** (LLM-enhanced swarm) | Future extension. MetricsSpace provides the observation data that LLM agents use for sophisticated role adaptation and team coordination. |

## Decisions

This spec implements the following design decisions:

| Decision | Title |
|----------|-------|
| D73 | Swarm architecture — extend stigmergy, not separate strategy |
| D74 | Role emergence — multi-dimensional behavioral fingerprinting |
| D75 | Team model — signal-based affinity clusters |
| D76 | Work redistribution — departure event + rule reaction |
| D77 | Swarm progress tracking — exploration, consensus, stability metrics |
| D78 | MetricsSpace — 5th WorkerRuntime facet |
| D79 | SwarmConfig — inside StigmergyConfig, presence-activated |
| D80 | Detection frequency — periodic + event-triggered hybrid |
| D81 | Swarm event types — 6 new CaseHubEventType values |
| D82 | Module placement — extend existing stigmergy packages |
| D83 | Pipeline decomposition — CaseEvaluationPipeline with phase handlers |

Cross-references to foundation SPI decisions: D1-D9 (observation), D10-D18 (signals), D19-D28 (interests/facets), D29-D31 (coordination state), D32-D36 (neighbors), D37-D44 (local rules), D45-D59 (convergence), D60-D72 (stigmergy execution model).

## References

- `StigmergyCoordinator.java` — agent lifecycle tracking, pattern detection
- `StigmergyStrategy.java` — named planning strategy, first-cycle dispatch
- `StigmergyConfig.java` — existing config records (StigmergyDefaults, CoordinationConfig)
- `ObservationRegistry.java` — observer storage, per-agent interest key queries
- `SignalRegistry.java` — signal storage, `sources` tracking, `consensusSignals()`
- `RuleRegistry.java` — rule storage, per-cycle firing records
- `ActivityTracker.java` — sliding-window rate computation, cumulative counters
- `WorkerRuntime.java` — api coordination surface (facets: signals, interests, neighbors, rules)
- `DefaultWorkerRuntime.java` — runtime-core WorkerRuntime implementation
- `WorkerRuntimeFactory.java` — factory wiring registries into runtime
- `CaseContextChangedEventHandler.java:1384-1425` — convergenceDetection phase
- `CaseStatusChangedHandler.java` — terminal state cleanup
- `CaseHubEventType.java` — existing event type enum
- `NeighborSpace.java` — neighbor queries (shared interests, shared signals, complementary)
- `Neighbor.java`, `NeighborRelation.java` — neighbor data model
- `Resettable` interface — demo/test replay
- 2026-09-18-stigmergy-execution-model-design.md — stigmergy spec (#1111)
- arXiv:2603.28990 ("Drop the Hierarchy and Roles" — 5,006 emergent roles)
- arXiv:2510.10047 (SwarmSys — Explorer/Worker/Validator cycle)
- arXiv:2504.00587 (AgentNet — dynamic DAG topology)
- engine#1104 — Hive Mind epic
- engine#1112 — this issue
- engine#1113 — self-provisioning (future consumer of role data)
- D73-D83 — design decisions
