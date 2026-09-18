# D73 Exploration: Role Emergence — Behavioral Fingerprinting

## First Principles

### What is a "role" in a self-organizing system?

In biological swarms, roles aren't assigned — they emerge from four observable dimensions:

1. **What the agent responds to** (stimulus sensitivity / perception domain)
2. **What the agent does** (behavioral patterns / action domain)
3. **How the agent affects the environment** (consequences)
4. **How other agents respond** (social feedback)

The "Drop the Hierarchy and Roles" paper found agents spontaneously created 5,006 unique roles from 8 agents. The key insight: roles weren't static labels — they were *behavioral trajectories* that evolved over time. An agent might start as an "explorer" and gradually become a "specialist."

The ant response threshold model shows that roles emerge from *differential sensitivity* — ants with lower thresholds for a stimulus respond first, naturally allocating tasks without any assignment mechanism. In CaseHub, agents with different interest thresholds and rule conditions naturally specialize.

### What data does CaseHub already have?

Each agent in the stigmergy model leaves observable traces across the existing registries:

| Registry | Static features (what agent CAN do) | Dynamic features (what agent IS doing) |
|----------|--------------------------------------|---------------------------------------|
| ObservationRegistry | Registered interest keys, interest types | — (observations are per-cycle, ephemeral) |
| SignalRegistry | — | Signal names deposited, deposit frequency, reinforcement counts |
| RuleRegistry | Registered rule IDs, condition types, action types | Which rules fire, firing frequency |
| (Rule actions) | — | Context keys written, signal names deposited from rules |

**Key observation:** The engine already has COMPLETE behavioral information. Role detection is a QUERY over existing data, not new data collection. This aligns perfectly with the "tracking + detection only" scope.

### Two classes of features

**Static features** — from registrations (what the agent is set up to do):
- Interest keys registered (perception domain)
- Rule IDs registered (decision repertoire)
- Action types in rules (action capabilities)

These change infrequently. An agent registers interests and rules during `execute()` and rarely modifies them.

**Dynamic features** — accumulated over time (what the agent actually does):
- Signal deposit counts per signal name
- Rule firing counts per rule
- Context write counts per key

These evolve continuously. An agent's behavioral fingerprint SHOULD change as it adapts.

The combined fingerprint captures both: same static features but different dynamic patterns = same capabilities, different behavior = different roles.

## Representation Analysis

### Approach A: Set intersection (Jaccard)

Fingerprint = set of feature names (presence only).
Similarity = |A ∩ B| / |A ∪ B|.

**Strengths:**
- Simplest to implement
- No normalization needed
- Binary — easy to explain

**Weaknesses:**
- Loses intensity: agent depositing "overheating" 100 times looks identical to one depositing it once
- Can't distinguish specialists from generalists
- Static features dominate (registered once, always present)

### Approach B: Feature vector + cosine similarity

Fingerprint = sparse Map<String, Double> (feature → weight).
Similarity = dot(A,B) / (|A| × |B|).

**Strengths:**
- Captures intensity (100 deposits ≠ 1 deposit)
- Magnitude-invariant (high-activity and low-activity agents comparable)
- Natural for mixed static/dynamic features
- Well-understood mathematically

**Weaknesses:**
- Requires normalization strategy (how to weight static vs. dynamic?)
- Cosine similarity on very sparse high-dimensional vectors can be noisy
- Need to accumulate dynamic counts — RuleRegistry only stores last-cycle firings

### Approach C: Multi-dimensional fingerprint (recommended synthesis)

Fingerprint = structured record with four domain-specific sub-vectors, each a sparse Map<String, Double>:

```
BehavioralFingerprint:
  perception:  {interest_key → 1.0}          // binary, from ObservationRegistry
  communication: {signal_name → normalized_deposit_count}  // from SignalRegistry
  decision:    {rule_id → normalized_firing_count}          // accumulated by tracker
  effect:      {context_key → normalized_write_count}       // accumulated by tracker
```

Similarity = weighted average of per-domain cosine similarities:

```
similarity = w_p × cos(A.perception, B.perception)
           + w_c × cos(A.communication, B.communication)
           + w_d × cos(A.decision, B.decision)
           + w_e × cos(A.effect, B.effect)
```

Default weights: equal (0.25 each). Configurable via SwarmConfig.

**Why four domains, not one flat vector?**

1. **Different semantics:** perception features are binary (presence), communication features are normalized counts. Mixing them in one vector requires careful normalization that obscures the domain structure.

2. **Domain-weighted similarity:** a case author can say "roles are defined by what agents communicate, not what they watch" by increasing the communication weight. This is impossible with a flat vector.

3. **Per-domain analysis:** you can report "agents A and B have identical perception but divergent effects" — which is more actionable than "similarity = 0.6."

4. **Sparse efficiency:** each domain vector is very sparse (≤20 interests, ≤100 signals, ≤50 rules). Per-domain cosine on small sparse vectors is faster than one large sparse vector.

**Why weighted cosine, not concatenated cosine?**

Concatenated: merge all domains into one vector, compute one cosine.
Problem: domains with more features dominate. If an agent has 20 interest keys and 2 signal names, the perception domain swamps the communication domain.

Weighted average of per-domain cosines treats each domain as an independent axis. The weights control the *importance* of each domain, not the *size*.

## Clustering Analysis

For role cluster detection, three algorithms are viable at CaseHub's scale (N ≤ 20 agents):

### Connected components (single-linkage)

Build graph: edge between agents with similarity > threshold.
Find connected components. Each component ≥ minSize is a role.

**Risk:** chain effect — A~B and B~C clusters even if A and C are dissimilar.
**Mitigation:** additionally require average internal similarity > threshold.

### Average-linkage agglomerative

Merge closest clusters until no merge exceeds threshold.
Each remaining cluster ≥ minSize is a role.

More robust than single-linkage. No chain effect.
O(N³) worst case but N ≤ 20 makes this negligible.

### Connected components with internal average check (recommended)

Hybrid: connected components for candidate clusters, then verify average internal similarity.

1. Build adjacency graph (sim > threshold)
2. Find connected components
3. For each component, compute average pairwise similarity
4. If average < threshold, split by removing weakest edge and recurse
5. Components ≥ minSize with avg similarity ≥ threshold are role clusters

Simple, deterministic, handles chain effects.

## Accumulation Strategy

Dynamic features (rule firings, context writes) are ephemeral in the registries. The RoleTracker needs to accumulate them.

**Sliding window accumulation:**
- Per-agent, per-feature: maintain a count over the last N evaluation cycles
- On each detection cycle: add current-cycle counts, subtract oldest cycle's counts
- Normalize by total per-agent activity within the window

Window size = `roleDetectionWindow` in SwarmConfig (default: 20 cycles).

This gives a recent-behavior fingerprint that naturally adapts as agent behavior evolves. Old behavioral patterns decay out of the window.

**Storage:** Per-agent circular buffer of per-cycle feature maps. Bounded by agents × window_size × features_per_cycle. With 20 agents × 20 cycles × 50 features = 20,000 entries. Negligible.

## Role Evolution Detection

Compare current role cluster set to previous detection cycle's set:

| Event | Condition |
|-------|-----------|
| SWARM_ROLE_EMERGED | New cluster detected (no matching cluster in previous set) |
| SWARM_ROLE_DISSOLVED | Previous cluster no longer exists |
| SWARM_ROLE_SHIFT | Agent moved from one cluster to another |
| SWARM_ROLE_CONVERGED | Two previous clusters merged into one |
| SWARM_ROLE_SPECIALIZED | One previous cluster split into two |

Cluster matching: two clusters match if they share > 50% of their members (majority overlap).

## Recommendation

**Approach C (multi-dimensional fingerprint)** is the strongest design because:
1. It captures all four behavioral dimensions that define a role
2. Domain weights give case authors control over what "role" means in their domain
3. Per-domain analysis gives richer audit data than a single similarity score
4. Sliding window accumulation captures behavioral evolution naturally
5. It's built entirely from existing registry queries — no new data collection
