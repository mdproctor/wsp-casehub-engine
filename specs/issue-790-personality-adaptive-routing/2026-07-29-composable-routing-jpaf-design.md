# Composable Routing Architecture + JPAF Personality-Adaptive Routing

**Epic:** engine#790 — Personality-adaptive agent routing
**Date:** 2026-07-29
**Covers:** #791, #792, #793, #794, #795, #796
**Reference:** [JPAF paper](https://arxiv.org/abs/2601.10025) — Structured Personality Control and Adaptation for LLM Agents

## Summary

Two interleaved changes:

1. **Composable routing signal architecture** — replaces the mutually exclusive `AgentRoutingStrategy` model at Layer 3 (Classical Strategy Execution) with a compositor that blends scores from independent signal providers. Evolves the existing `RoutingSignalProvider` SPI and `RoutingSignalAssembler` to support hard exclusion, escalation, and weighted composition. Layer 4 strategies (AI-driven: `LlmAgentRoutingStrategy`, `CbrAgentRoutingStrategy`) remain as `AgentRoutingStrategy` implementations.

2. **JPAF personality-adaptive routing** — a new signal provider that scores candidates by cognitive function alignment between task demand profiles and agent personality profiles, with reinforcement-compensation signal recording and threshold-triggered reflection for personality evolution.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Cognitive function determination | **(D) Hybrid** — demand profiles on capabilities for routing, agent's dom/aux for signal attribution | Routing needs task-side data at dispatch time; reinforcement captures what the agent actually engaged |
| TemporaryWeight storage | **No materialized state** — effective weights computed at probe time from `DispositionSignalStore` activation counts + base weights | TemporaryWeight = BaseWeight + count × Δw is derivable; no new store SPI needed |
| Decay mechanism | **Multiplicative decay** via `DispositionSignalStore.decay(agentId, tenancyId, decayFactor)` using `DECAY_FACTOR = 0.20` from `JungianFunctionTerm` | Preserves signal direction at reduced magnitude; a function activated 20 times decays to ~4 — still detectable, still influencing routing. Consistent with existing `DefaultDispositionEvolution` which returns `Dampened(decayFactor)` and uses `decay()` not `clear()` on evolution |
| Responsibility split | **Engine** records signals + routes; **eidos** computes effective weights + evaluates reflection + updates descriptors | Personality theory (decision rules, Jungian validation) belongs in eidos; runtime orchestration belongs in engine |
| Routing composition | **Evolve existing `RoutingSignalProvider` SPI** — extend `CandidateSignal` from a score-only record to a sealed interface with Score/Exclude/Escalate; compositor blends with configurable weights | Builds on existing SPI already used by blocks strategies; avoids creating a parallel interface with the same name |
| Blocks-layer scope | **Layer 4 strategies remain as `AgentRoutingStrategy` implementations** — composable architecture replaces Layer 3 only | LLM and CBR strategies make AI-driven selection decisions, not just scoring. They are architecturally different from signal providers and continue to override the compositor via `@Priority` |

---

## Part 1: Composable Routing Signal Architecture

### Existing SPI Evolution

The codebase already has `RoutingSignalProvider` (in `io.casehub.api.spi.routing`) with `RoutingSignalAssembler` composition. The existing SPI returns `RoutingSignal(Map<String, CandidateSignal>)` where `CandidateSignal` is `(double score, @Nullable String reason)` — a supplementary enrichment mechanism used by blocks strategies.

This spec evolves that SPI into the **primary scoring mechanism** for Layer 3 routing:

### RoutingSignalProvider SPI (evolved)

**Package:** `io.casehub.api.spi.routing` (engine-api) — same package, same interface name

```java
public interface RoutingSignalProvider extends NamedStrategy {
    @Nullable RoutingSignal evaluate(AgentRoutingContext context, List<AgentCandidate> eligible);
}
```

Changes from current SPI:
- Extends `NamedStrategy` (compatible — `id()` already exists on current interface)
- Method renamed from `signal()` to `evaluate()` (breaking, mechanical migration)
- `CandidateSignal` evolves from record to sealed interface (see below)

### RoutingSignal (evolved)

```java
public record RoutingSignal(Map<String, CandidateSignal> candidates) {

    public sealed interface CandidateSignal {
        record Score(double value, @Nullable String rationale) implements CandidateSignal {}
        record Exclude(String reason) implements CandidateSignal {}
        record Escalate(EscalationReason reason, String rationale) implements CandidateSignal {}
    }
}
```

- **Score** — normalized 0.0–1.0 value with rationale
- **Exclude** — hard gate, candidate removed from consideration
- **Escalate** — candidate removed with escalation flag; if ALL candidates are removed and any was Escalate, the compositor returns `RoutingResult.Escalated`
- **Absent from map** — provider has no data for this candidate; weight redistributed among providers that returned Score for this candidate (preserves existing sparse-map semantics)

### ComposableAgentRoutingStrategy

**Package:** `io.casehub.engine.internal.routing` (engine runtime)
**CDI:** `@DefaultBean @ApplicationScoped @Unremovable`, id=`"composable"`

Replaces `LeastLoadedAgentStrategy` as the default Layer 3 strategy. Injects `RoutingSignalAssembler` for provider discovery and signal collection; implements weighted scoring on the assembler's output.

**Scoring algorithm:**

1. For each candidate, collect `CandidateSignal` from all configured providers
2. If ANY provider returns `Exclude` → remove candidate
3. If ANY provider returns `Escalate` → remove candidate, flag escalation reason
4. For remaining candidates: redistribute weight of absent providers among providers that returned `Score` for that candidate
5. If ALL providers are absent for a candidate (no Score entries) → assign neutral score (0.5)
6. Compute weighted sum of `Score` values with normalized weights
7. Select highest-scoring candidate → `RoutingResult.Selected`
8. If all candidates removed AND any was `Escalate` → `RoutingResult.Escalated` with first escalation reason
9. If all candidates removed AND no `Escalate` → `RoutingResult.Unresolvable` with aggregated reasons

**Weight configuration:**

- Default: equal weights across all discovered providers
- Per-case override: `CaseDefinition.routingSignalWeights` (`Map<String, Double>`, nullable)
- `AgentRoutingContext.routingSignalWeights` carries the per-case override
- When `routingSignalWeights` is set, only named providers are called

### Signal Provider Decomposition

| Provider | id | Module | Scoring | Exclude | Escalate | Absent when |
|----------|-----|--------|---------|---------|----------|------------|
| `WorkloadSignalProvider` | `"workload"` | engine runtime | `1.0 / (1.0 + runningJobs)` | Never | Never | Never |
| `TrustSignalProvider` | `"trust"` | engine-ledger | Trust score for QUALIFIED; availability for BOOTSTRAP | Phase 2b/3 | `BORDERLINE_STALEMATE` (all borderline), `NO_QUALIFIED_AGENT` (bootstrap only + policy requires) | When trust data unavailable |
| `ExperienceSignalProvider` | `"experience"` | engine runtime | `ExperienceAnalyser.workerSuccessRates()` | Never | Never | When no experiences |
| `PersonalitySignalProvider` | `"personality"` | engine runtime | Cosine similarity (effective weights vs demand profile) | Never | Never | When agent has no disposition profile or capability has no demand profile |
| `SemanticSignalProvider` | `"semantic"` | engine-ai | Embedding similarity | Never | Never | When embedding service unavailable |

Note: `ExperienceSignalProvider` (formerly `CbrSignalProvider`) uses `ExperienceAnalyser.workerSuccessRates()` — a lightweight success-rate lookup already used as a secondary blend inside `TrustWeightedAgentStrategy`. This is distinct from blocks' `CbrAgentRoutingStrategy`, which performs full CBR retrieval with feature extraction, graph query fallback, and signal enrichment.

**Classpath-driven auto-enrichment:**
- Engine runtime only → workload → equivalent to old LeastLoaded
- + engine-ledger → workload + trust → equivalent to old TrustWeighted
- + engine-ai → workload + trust + semantic
- + personality config → workload + trust + personality + experience

### Blocks-Layer Strategies (Layer 4)

`LlmAgentRoutingStrategy` (`@Priority(100)`) and `CbrAgentRoutingStrategy` (`@Priority(101)`) in casehub-blocks remain as `AgentRoutingStrategy` implementations. When blocks is on the classpath, these override `ComposableAgentRoutingStrategy` via CDI priority — the same mechanism used today.

Layer 4 strategies are architecturally different from signal providers:
- They make AI-driven selection decisions (LLM invocation, CBR retrieval + graph query)
- They compose trust classification internally via `RoutingSupport.applyTrustFilter()`
- `CbrAgentRoutingStrategy` already consumes `RoutingSignalProvider` output via `RoutingSignalAssembler` for enrichment — this continues to work with the evolved SPI

Supporting SPIs used by blocks strategies (`RoutingPromptSection`, `RoutingPromptAssembler`) are unaffected. New signal providers (personality, experience) are automatically available to blocks strategies via the existing `RoutingSignalAssembler` composition.

### AgentRoutingContext Changes

```java
public record AgentRoutingContext(
    UUID caseId,
    String capabilityName,
    JsonNode caseContext,
    String tenancyId,
    List<RetrievedExperience> experiences,
    CognitiveDemand cognitiveDemand,            // NEW — nullable, from Capability
    Map<String, Double> routingSignalWeights) {} // NEW — nullable, from CaseDefinition
```

Constructed by `CaseContextChangedEventHandler.publishWorkerSchedule()` from `CaseDefinition` and `Capability`, both already available at the call site.

### Migration

**Deleted:**
- `LeastLoadedAgentStrategy` → replaced by `WorkloadSignalProvider`
- `TrustWeightedAgentStrategy` → replaced by `TrustSignalProvider`
- `SemanticAgentRoutingStrategy` → replaced by `SemanticSignalProvider`
- `TrustCandidateClassifier` phase classification logic → moves into `TrustSignalProvider`

**Preserved:**
- `LlmAgentRoutingStrategy` (blocks) — unchanged
- `CbrAgentRoutingStrategy` (blocks) — unchanged; `RoutingSignalAssembler` calls updated from `signal()` to `evaluate()`
- `RoutingPromptSection`, `RoutingPromptAssembler` — unchanged
- `RoutingSignalAssembler` — stays as discovery + assembly + validation layer (SRP); the compositor handles weighted scoring on its output. Evolution scope: `clampScores()` pattern-matches on the sealed `CandidateSignal` — clamp only `Score.value` to [0.0, 1.0], pass through `Exclude` and `Escalate` unchanged

**YAML migration:**
- `agentRouting: "least-loaded"` → delete the line
- `agentRouting: "trust-weighted"` → delete the line
- `agentRouting: "semantic"` → delete the line, optionally add `routingSignalWeights`

**EngineStrategyResolver:** No constructor changes needed. `ComposableAgentRoutingStrategy` is registered as `@DefaultBean` for `AgentRoutingStrategy`, discovered by the resolver's existing `@Any Instance<AgentRoutingStrategy>`. Signal providers are discovered by `RoutingSignalAssembler` (existing CDI discovery), not by the strategy resolver. Since `RoutingSignalProvider extends NamedStrategy`, providers are also visible through the resolver's `@Any Instance<NamedStrategy>` catch-all for diagnostic/introspection purposes, but the compositor accesses them exclusively through the assembler.

**ADR-0003:** Records the (now superseded) decision to make `AgentRoutingStrategy.select()` return `Uni<AgentAssignment>`. The current SPI returns `RoutingResult` synchronously. Mark ADR-0003 as Superseded — virtual threads removed the reactive requirement.

---

## Part 2: JPAF Personality-Adaptive Routing

### CognitiveDemand

**Package:** `io.casehub.api.model` (engine-api), field on `Capability`

```java
public record CognitiveDemand(Map<String, Double> functionWeights) {
    public CognitiveDemand {
        functionWeights = Map.copyOf(functionWeights);
        double sum = functionWeights.values().stream()
            .mapToDouble(Double::doubleValue).sum();
        if (Math.abs(sum - 1.0) > 0.01) {
            throw new IllegalArgumentException(
                "functionWeights must sum to 1.0, got " + sum);
        }
    }
}
```

Keys are Jungian function name strings (Ti, Te, Fi, Fe, Si, Se, Ni, Ne) matching `AgentDisposition.dispositionProfile()` terms. String-typed to keep the engine decoupled from eidos vocabulary specifics. Nullable on `Capability` — capabilities without demand profiles are invisible to personality routing.

**YAML:**
```yaml
capabilities:
  - name: code-review
    cognitiveDemand:
      Ti: 0.6
      Ne: 0.3
      Si: 0.1
```

### PersonalitySignalProvider

**Package:** `io.casehub.engine.internal.routing` (engine runtime)
**CDI:** `@ApplicationScoped`, id=`"personality"`

```
evaluate(context, eligible):
  for each candidate:
    1. candidate.agentDescriptor() == null → absent
    2. dispositionProfile is empty → absent
    3. context.cognitiveDemand() == null → absent
    4. Call DispositionHealth.probe(descriptor, probeContext) → effectiveWeights
    5. Compute cosine similarity between effectiveWeights and demand profile
    6. Return Score(similarity, rationale)
  return RoutingSignal with per-candidate results (absent candidates omitted from map)
```

**Cosine similarity:** Both vectors are 8-dimensional (one per Jungian function, zero-filled for missing entries). Non-negative vectors → result is 0.0–1.0.

### Signal Recording

**PersonalitySignalRecorder** — `@ApplicationScoped` in engine runtime, injected into `WorkflowExecutionCompletedHandler`.

Called after outcome processing:
```java
personalitySignalRecorder.record(caseInstance, workerName, capabilityName, outcome);
```

**Recording logic:**

```
record(caseInstance, workerName, capabilityName, outcome):
  1. Look up AgentDescriptor via CaseDefinition.agentDescriptorFor(workerName)
     → null? return
  2. Extract dominant + auxiliary from dispositionProfile (top 2 by weight)
  3. Look up Capability → get CognitiveDemand
     → null? return
  4. Signal attribution:

  SUCCESS:
    engagedFunction = whichever of {dominant, auxiliary} has higher weight
                      in the demand profile
    dispositionSignalStore.recordActivation(agentId, tenancyId, engagedFunction)

  DECLINE / FAILURE / EXPIRED:
    compensatoryFunction = highest-demand function NOT in {dominant, auxiliary}
    dispositionSignalStore.recordActivation(agentId, tenancyId, compensatoryFunction)
```

The compensatory function receives an activation signal — it represents "this function was activated as compensation." This follows the JPAF paper: compensation activates a non-dom/aux function, growing its effective weight toward potential restructuring. Only the compensatory function is recorded; the engaged function receives no signal. The `DispositionSignalStore` API supports positive activation counts only — the JPAF mechanism drives evolution through compensation growth, not direct punishment of the engaged function.

**Edge cases:**
- Demand profile has no weight for dom or aux → use highest-demand function overall
- All demand on dom/aux (no outside functions) → skip compensation recording on failure
- Tied functions → pick alphabetically first (deterministic)

### Effective Weight Computation

**Owner:** eidos-runtime (`DispositionHealth` implementation)

Effective weights are computed at probe time — no materialized TemporaryWeight state:

```
effectiveWeight(f) = baseWeight(f) + activationCount(f) × Δw
```

Where:
- `baseWeight(f)` comes from `AgentDisposition.dispositionProfile()`
- `activationCount(f)` comes from `DispositionSignalStore.activationCounts(agentId, tenancyId)` → `Map<String, Integer>`, looked up by function term
- `Δw = 0.06` (`JungianFunctionTerm.REINFORCEMENT_DELTA`)

`DispositionHealth.probe()` returns:
- `Aligned(effectiveWeights)` — no threshold crossed
- `Drifted(effectiveWeights, mostActivated, driftMagnitude)` — drift detected but below threshold
- `EvolutionPending(type, candidateFunction, effectiveWeights)` — threshold crossed, reflection needed

### Reflection Trigger

**Flow:**

```
Worker completes task
  → WorkflowExecutionCompletedHandler processes outcome
  → PersonalitySignalRecorder records activation signal
  → Recorder calls DispositionHealth.probe()
  → If EvolutionPending:
      → Call DispositionEvolution.evaluate(descriptor, evolutionPending)
      → If Evolved(newProfile, previousLabel, newLabel):
          → Re-register AgentDescriptor with newProfile via AgentDescriptorRegistrar
          → Log evolution to EventLog
      → If Dampened(decayFactor):
          → Call dispositionSignalStore.decay(agentId, tenancyId, decayFactor)
          → Log dampening to EventLog
  → If Aligned or Drifted: done
```

`DispositionEvolution` is an existing eidos SPI — no new SPI is needed:

```java
public interface DispositionEvolution {
    EvolutionResult evaluate(AgentDescriptor descriptor,
                             DispositionHealth.DispositionStatus.EvolutionPending pending);

    sealed interface EvolutionResult {
        record Evolved(List<DispositionValue> newProfile,
                       String previousTypeLabel, String newTypeLabel) implements EvolutionResult {}
        record Dampened(double decayFactor) implements EvolutionResult {}
    }
}
```

`DefaultDispositionEvolution` already implements all 4 JPAF decision rules. On `Evolved`, it internally calls `signalStore.decay()` to reduce accumulated activation counts while preserving signal direction. On `Dampened`, the caller handles the decay.

**The 4 JPAF decision rules** (evaluated by `DispositionHealth.probe()`, applied by `DefaultDispositionEvolution`):

1. **Dominant-Auxiliary Role Swap** — auxiliary's effective weight exceeds dominant's → swap roles
2. **Dominant Replacement** — a function sharing the dominant's attitude (E/I) but opposite function type (judging↔perceiving) has effective weight ≥ dominant's base weight → replaces dominant
3. **Auxiliary Replacement** — same pattern for auxiliary
4. **Structural Reorganization** — a function outside the dom/aux pair exceeds dominant's base weight → becomes new dominant; compatible auxiliary selected per Jungian pairing rules

Evaluation ordering is an eidos implementation detail in `DispositionHealth.probe()`, which determines which `EvolutionType` to return in the `EvolutionPending` status. `DefaultDispositionEvolution` receives a single type and applies the corresponding transformation.

**Synchronous execution:** Runs inline on the completion handler's virtual thread. Operations are lightweight.

**Concurrency:** When an agent completes two tasks concurrently, both virtual threads execute the record → probe → evaluate → re-register sequence without synchronization. This is intentionally benignly racy — personality evolution is probabilistic, not transactional. The worst case is two consecutive evolutions from concurrent completions: Thread A evolves the personality and decays signals; Thread B sees the new profile with partially-stale counts and may trigger a second evolution. This is self-correcting — the second evolution is based on valid (if slightly stale) data, and the next probe after both complete reflects the stable state. Per-agent locking would impose transactional semantics on an inherently probabilistic system with no correctness benefit.

**Routing path is unaffected:** `PersonalitySignalProvider.evaluate()` reads effective weights from `probe()` but never triggers reflection. `EvolutionPending` still provides valid effective weights for scoring.

---

## Issue Decomposition

### Engine issues (composable architecture)

| ID | Title | Depends on | Scale |
|----|-------|-----------|-------|
| NEW-1 | Evolve `RoutingSignalProvider` SPI — `CandidateSignal` sealed interface, `evaluate()` method, `NamedStrategy` extension | — | M |
| NEW-2 | `ComposableAgentRoutingStrategy` — compositor with weight config, Escalate handling, all-abstain neutral | NEW-1 | M |
| NEW-3 | `WorkloadSignalProvider` + `ExperienceSignalProvider` | NEW-1 | S |
| NEW-4 | `TrustSignalProvider` (refactor `TrustWeightedAgentStrategy`) — includes Escalate signals | NEW-1 | M |
| NEW-5 | `SemanticSignalProvider` (refactor `SemanticAgentRoutingStrategy`) | NEW-1 | S |
| NEW-6 | `AgentRoutingContext` expansion + `CaseDefinition.routingSignalWeights` + YAML mapping | NEW-1 | S |
| NEW-7 | Delete old strategies + update `EngineStrategyResolver` + blocks `RoutingSignalAssembler` migration (`signal()` → `evaluate()`) | NEW-3, NEW-4, NEW-5, NEW-6 | S |
| NEW-8 | Mark ADR-0003 as Superseded | — | XS |

### JPAF issues (remapped)

| Original | New scope | Depends on |
|----------|-----------|-----------|
| #791 | `PersonalitySignalRecorder` in `WorkflowExecutionCompletedHandler` — uses `DispositionSignalStore.recordActivation()` | eidos#115 (landed) |
| #792 | No engine code — effective weights computed by eidos `DispositionHealth` impl using `DispositionSignalStore.activationCounts()` | EIDOS-A |
| #793 | Engine: reflection trigger in recorder, calls `DispositionEvolution.evaluate()`, handles `Evolved`/`Dampened` | #791 |
| #794 | `PersonalitySignalProvider` in composable architecture | NEW-1, NEW-6 |
| #795 | `CognitiveDemand` on `Capability` + cosine similarity | #794 |
| #796 | No engine code — decay handled by `DispositionEvolution` (Evolved: internal decay; Dampened: caller calls `decay()`) | #793 |

### Eidos issues (to file)

| ID | Title |
|----|-------|
| EIDOS-A | `DispositionHealth` implementation — effective weight computation from `DispositionSignalStore.activationCounts()`, `EvolutionPending` detection with all 4 JPAF rule evaluations |
| EIDOS-C | Over-concentration trigger in `DispositionHealth.probe()` — dominant function's base weight exceeding 0.5 should trigger `EvolutionPending` per JPAF |

Note: EIDOS-B (`PersonalityReflectionEngine`) is removed — `DispositionEvolution` and `DefaultDispositionEvolution` already exist with the 4 JPAF decision rules implemented.

### Execution order

```
Phase 1: NEW-1 (SPI evolution)
Phase 2: NEW-2 (compositor)
Phase 3: NEW-3, NEW-4, NEW-5, NEW-6 (parallel — providers + config)
Phase 4: NEW-7 (delete old strategies, wire everything, blocks migration)
Phase 5: #791 (signal recording)
Phase 6: #794 + #795 (personality provider + alignment scoring)
Phase 7: #793 engine-side (reflection trigger)
Eidos:   EIDOS-A, EIDOS-C (parallel with engine phases 1–4)
```

Phases 1–4 deliver the composable architecture. Phases 5–7 layer JPAF on top.

---

## YAML Full Example

```yaml
name: aml-investigation
namespace: aml

signals:
  - name: sar-filing
    contextType: io.casehub.aml.model.SarReport

capabilities:
  - name: entity-resolution
    inputProjection: ".transaction"
    cognitiveDemand:
      Ti: 0.5
      Si: 0.3
      Ne: 0.2

  - name: risk-assessment
    inputProjection: ".entityProfile"
    cognitiveDemand:
      Ni: 0.4
      Te: 0.3
      Fi: 0.2
      Se: 0.1

routingSignalWeights:
  trust: 0.3
  personality: 0.4
  workload: 0.2
  experience: 0.1

workers:
  - name: entity-resolver
    capabilityName: entity-resolution
    agent:
      model: openai/gpt-4
      systemPrompt: "..."
```
