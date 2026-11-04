# AgentDecisionLoop — Unified Observe→Reason→Act Primitive

**Issue:** agency#379
**Parent epic:** agency#378 (Agency Intelligence Layer)
**Cross-repo parent:** agency#377 (Distributed Per-Agent LLM Coordination)
**Depends on:** engine#1282 (coordination vocabulary types) — done, engine#1285 (work lifecycle) — done

## Summary

The fundamental primitive for all LLM coordination decisions. One agent's reasoning
cycle — observe environment, reason via LLM, produce a decision — composed from
existing stigmergy primitives (ContentSummariser, ChangeGatedEmissionPolicy,
InFlightGate, SwarmPromptSection) plus engine's coordination vocabulary
(DecisionScope, ObservationStrategy, CommunicationMode, AgentWorkState).

The centralised coordinator and per-agent local loops are the same abstraction,
differently parameterised. A decision loop is a **playbook** — a declarative
coordination recipe. The Java types are designed for YAML parity (#385) but this
issue delivers Java only.

The decision loop sits below the agentic execution model (ExecutionDriver,
DecompositionStrategy, AggregationStrategy). It is the per-agent reasoning
primitive that topology implementations compose. The agentic framework
orchestrates multi-agent workflows; the decision loop handles one agent's
observe→reason→decide cycle within that workflow.

## Core Concept

Two types separate configuration from execution:

- **`DecisionLoopDescriptor`** — the playbook. Immutable data record describing
  observation strategy, decision scope, communication mode, budget policy, and
  prompt section names. YAML-serialisable (no lambdas, no callbacks).
- **`AgentDecisionLoop`** — the runtime. Executes a descriptor for a specific
  agent. Internally composes a `SummarisationRunner` for the observe→reason
  pipeline and outputs `DecisionOutcome` for the caller to execute.

A `DecisionLoopFactory` assembles the runtime from a descriptor by resolving
prompt section names from a `PromptSectionRegistry`, constructing the appropriate
emission policy from `ObservationStrategy`, and wiring the budget tracker.

## DecisionLoopDescriptor

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public record DecisionLoopDescriptor(
    String name,
    ObservationStrategy observationStrategy,
    DecisionScope scope,
    CommunicationMode communicationMode,
    AgentBudgetPolicy budgetPolicy,
    List<String> promptSectionNames
) {
    public DecisionLoopDescriptor {
        Objects.requireNonNull(name, "name must not be null");
        Objects.requireNonNull(observationStrategy, "observationStrategy must not be null");
        Objects.requireNonNull(scope, "scope must not be null");
        Objects.requireNonNull(communicationMode, "communicationMode must not be null");
        Objects.requireNonNull(budgetPolicy, "budgetPolicy must not be null");
        promptSectionNames = promptSectionNames != null
            ? List.copyOf(promptSectionNames) : List.of();
    }
}
```

**Design properties:**
- Immutable — `List.copyOf` in compact constructor
- YAML-friendly — all fields are enums, records, or string lists
- No runtime dependencies — pure data, no behaviour beyond validation
- `name` identifies the playbook for logging, events, and YAML file correlation

### Future YAML surface (#385)

```yaml
# agent-local.loop.yaml
name: agent-local
observation: EVENT_DRIVEN
scope: TASK_ADAPTATION
communication: STIGMERGY
budget:
  maxCallsPerAgent: 5
  maxTokensPerAgent: 10000
prompt-sections:
  - experience-replay
  - role-context
```

## AgentBudgetPolicy

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public record AgentBudgetPolicy(
    int maxCallsPerAgent,
    long maxTokensPerAgent
) {
    public AgentBudgetPolicy {
        if (maxCallsPerAgent < 0)
            throw new IllegalArgumentException("maxCallsPerAgent must be non-negative");
        if (maxTokensPerAgent < 0)
            throw new IllegalArgumentException("maxTokensPerAgent must be non-negative");
    }

    public static AgentBudgetPolicy defaults() {
        return new AgentBudgetPolicy(10, 50_000);
    }

    public static AgentBudgetPolicy unlimited() {
        return new AgentBudgetPolicy(Integer.MAX_VALUE, Long.MAX_VALUE);
    }
}
```

YAML-serialisable. The per-agent policy. Global governance is handled by the
composite `AgentBudgetTracker` (see below).

## DecisionContext

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public record DecisionContext(
    UUID caseId,
    String agentId,
    DecisionScope scope,
    CommunicationMode communicationMode,
    TopologyDescriptor topology
) {
    public DecisionContext {
        Objects.requireNonNull(caseId, "caseId must not be null");
        Objects.requireNonNull(agentId, "agentId must not be null");
        Objects.requireNonNull(scope, "scope must not be null");
        Objects.requireNonNull(communicationMode, "communicationMode must not be null");
        Objects.requireNonNull(topology, "topology must not be null");
    }
}
```

Per-agent view of the coordination state. The centralised coordinator
constructs this from SwarmContext; distributed agents receive a lighter
view directly. The loop doesn't know about SwarmContext.

**Why `AgentWorkState` is excluded:** Work state is dynamic — it transitions
through IDLE → ASSIGNED → EXECUTING → IDLE over the loop's lifetime. Freezing
it in an immutable record at construction time produces stale data on subsequent
ticks. The `ContentSummariser` implementation can close over a live state
reference (e.g., a `Supplier<AgentWorkState>`) if it needs current work state
for prompt assembly.

**Relationship to SwarmContext:** SwarmContext is swarm-level shared mutable
state (roles, teams, progress, experiences). DecisionContext is a per-agent
snapshot derived from it. The mapping happens in the topology layer (#380/#381),
not in the loop.

## ObservationEvent

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public record ObservationEvent(
    String key,
    Object value,
    Instant timestamp
) {
    public ObservationEvent {
        Objects.requireNonNull(key, "key must not be null");
        Objects.requireNonNull(timestamp, "timestamp must not be null");
    }

    public static ObservationEvent of(String key, Object value) {
        return new ObservationEvent(key, value, Instant.now());
    }
}
```

The input type for the SummarisationRunner pipeline. Observation events are
keyed so `ChangeGatedEmissionPolicy` can watch specific keys (e.g.,
`"signal.quality"`, `"context.plan.*"`).

## DecisionOutcome

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public sealed interface DecisionOutcome {

    // Observation = io.casehub.api.spi.observation.Observation (engine-api)
    // — the same type used by EnvironmentWriter.storeObservations()
    record WriteBack(
        List<Observation> observations,
        List<SignalDeposit> signals,
        List<ContextWrite> contextWrites
    ) implements DecisionOutcome {
        public WriteBack {
            observations = observations != null ? List.copyOf(observations) : List.of();
            signals = signals != null ? List.copyOf(signals) : List.of();
            contextWrites = contextWrites != null ? List.copyOf(contextWrites) : List.of();
        }
    }

    record Route(RoutingDecision decision) implements DecisionOutcome {
        public Route {
            Objects.requireNonNull(decision, "decision must not be null");
        }
    }

    record Aggregate(AggregationResult result) implements DecisionOutcome {
        public Aggregate {
            Objects.requireNonNull(result, "result must not be null");
        }
    }

    record Composite(List<DecisionOutcome> outcomes) implements DecisionOutcome {
        public Composite {
            outcomes = List.copyOf(outcomes);
        }
    }

    record ScopeGated(
        DecisionType attempted,
        DecisionOutcome rejected
    ) implements DecisionOutcome {
        public ScopeGated {
            Objects.requireNonNull(attempted, "attempted must not be null");
            Objects.requireNonNull(rejected, "rejected must not be null");
        }
    }
}
```

### Variant semantics

| Variant | Semantics | Execution responsibility |
|---------|-----------|------------------------|
| WriteBack | Deposit observations, signals, context writes via EnvironmentWriter | Topology layer calls EnvironmentWriter |
| Route | Agent selection / team formation | Topology layer applies RoutingDecision |
| Aggregate | Multi-agent synthesis | Topology layer processes AggregationResult |
| Composite | One decision that produced multiple effects | Topology layer iterates and applies each |
| ScopeGated | LLM tried an action the scope doesn't permit | Logged for observability; not executed |

### Supporting records for WriteBack

```java
public record SignalDeposit(
    String name,
    double strength,
    @Nullable Duration halfLife
) {}

public record ContextWrite(
    String key,
    JsonNode value
) {}
```

These are data-only — they describe what should be written, not how. The
topology layer maps them to `EnvironmentWriter` calls with the appropriate
`caseId` and `agentId`.

## AgentBudgetTracker

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public class AgentBudgetTracker {

    private final AgentBudgetPolicy agentPolicy;
    private final @Nullable SwarmInvocationTracker sharedCap;
    private final AtomicInteger callCount = new AtomicInteger(0);
    private final AtomicLong tokenCount = new AtomicLong(0);

    public AgentBudgetTracker(AgentBudgetPolicy agentPolicy,
                              @Nullable SwarmInvocationTracker sharedCap) {
        this.agentPolicy = agentPolicy;
        this.sharedCap = sharedCap;
    }

    public AgentBudgetTracker(AgentBudgetPolicy agentPolicy) {
        this(agentPolicy, null);
    }

    public boolean tryAcquire() {
        int acquired = callCount.getAndIncrement();
        if (acquired >= agentPolicy.maxCallsPerAgent()) {
            callCount.decrementAndGet();
            return false;
        }
        if (sharedCap != null && !sharedCap.tryAcquire("agent-loop")) {
            callCount.decrementAndGet();
            return false;
        }
        return true;
    }

    public void recordTokens(long tokens) {
        tokenCount.addAndGet(tokens);
        if (sharedCap != null) {
            sharedCap.recordTokens(tokens);
        }
    }

    public boolean isBudgetExhausted() {
        return callCount.get() >= agentPolicy.maxCallsPerAgent()
            || tokenCount.get() >= agentPolicy.maxTokensPerAgent()
            || (sharedCap != null && sharedCap.isTokenBudgetExhausted());
    }

    public int callsUsed() { return callCount.get(); }
    public long tokensUsed() { return tokenCount.get(); }
}
```

Per-agent tracking with optional delegation to a shared swarm-level cap.
The per-agent tracker checks its own limit first, then the shared limit.
Thread-safe via atomic counters (same pattern as `SwarmInvocationTracker`).

## LoopPromptSection

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
@FunctionalInterface
public interface LoopPromptSection {
    String contribute(DecisionContext context);
}
```

The loop's own prompt section interface, parameterised by `DecisionContext`
(not `SwarmContext`). Topology implementations adapt existing `SwarmPromptSection`
instances when registering them:

```java
// In CentralisedTopology (#381) — adapting SwarmPromptSection to LoopPromptSection
SwarmPromptSection swarmSection = new SwarmExperienceSection(config);
LoopPromptSection adapted = ctx -> swarmSection.contribute(swarmContextFor(ctx));
registry.register("experience-replay", adapted);
```

This keeps the loop decoupled from swarm-specific context while allowing
existing prompt sections to be reused through adaptation.

## PromptSectionRegistry

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public interface PromptSectionRegistry {

    LoopPromptSection resolve(String name);

    default List<LoopPromptSection> resolveAll(List<String> names) {
        return names.stream().map(this::resolve).toList();
    }
}
```

Resolves prompt section names to `LoopPromptSection` instances. Implementations
are CDI/Spring beans. `resolve()` throws `IllegalArgumentException` for unknown
names — fail-fast at factory time, not at first tick.

### MapPromptSectionRegistry

A simple in-memory implementation for testing and programmatic use:

```java
public class MapPromptSectionRegistry implements PromptSectionRegistry {

    private final Map<String, LoopPromptSection> sections;

    public MapPromptSectionRegistry(Map<String, LoopPromptSection> sections) {
        this.sections = Map.copyOf(sections);
    }

    @Override
    public LoopPromptSection resolve(String name) {
        var section = sections.get(name);
        if (section == null) {
            throw new IllegalArgumentException(
                "Unknown prompt section: " + name
                + " — registered: " + sections.keySet());
        }
        return section;
    }
}
```

## DecisionLoopFactory

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public class DecisionLoopFactory {

    private final PromptSectionRegistry sectionRegistry;
    private final EventStreamBus<DecisionOutcome> outputBus;

    public DecisionLoopFactory(PromptSectionRegistry sectionRegistry,
                               EventStreamBus<DecisionOutcome> outputBus) {
        this.sectionRegistry = sectionRegistry;
        this.outputBus = outputBus;
    }

    public AgentDecisionLoop create(
            DecisionLoopDescriptor descriptor,
            DecisionContext context,
            ContentSummariser<ObservationEvent, DecisionOutcome> reasoner,
            @Nullable SwarmInvocationTracker sharedBudget) {

        var sections = sectionRegistry.resolveAll(descriptor.promptSectionNames());
        var budget = new AgentBudgetTracker(descriptor.budgetPolicy(), sharedBudget);

        // Wire prompt sections into the summariser — sections contribute
        // context that the reasoner uses during prompt assembly
        ContentSummariser<ObservationEvent, DecisionOutcome> withSections =
            new SectionAwareContentSummariser(reasoner, sections, context);

        // Wire budget gate — tryAcquire() before invocation
        ContentSummariser<ObservationEvent, DecisionOutcome> budgetAware =
            (events, prev) -> {
                if (!budget.tryAcquire()) {
                    return CompletableFuture.completedFuture(prev);
                }
                return withSections.summarise(events, prev);
            };

        var gate = new InFlightGate<>(budgetAware);

        EmissionPolicy<ObservationEvent, ?> emissionPolicy = buildEmissionPolicy(
            descriptor.observationStrategy());

        var runner = SummarisationRunner.<ObservationEvent, DecisionOutcome>builder(
                gate.summariser().asSummariser(),
                outputBus,
                DecisionLoopLevels.DECISION)
            .emissionPolicy(gate.gating(emissionPolicy))
            .build();

        return new AgentDecisionLoop(descriptor, context, runner, budget);
    }

    private EmissionPolicy<ObservationEvent, ?> buildEmissionPolicy(
            ObservationStrategy strategy) {
        return switch (strategy) {
            case EVENT_DRIVEN -> EmissionPolicy.window(WindowPolicy.ofCount(1));
            case TICK_DRIVEN -> EmissionPolicy.window(
                WindowPolicy.ofDuration(java.time.Duration.ofSeconds(5)));
            case DEMAND_DRIVEN -> (buffered, state, now) -> false; // never auto-emit
        };
    }
}
```

**EVENT_DRIVEN emission policy:** Uses `WindowPolicy.ofCount(1)` — emit on every
observation event. For loops that need selective triggering (fire only when
specific keys change), wrap with `ChangeGatedEmissionPolicy` in the topology
layer. Future descriptor versions may add a `watchedKeys` field to configure
this declaratively.

**Factory responsibilities:**
1. Resolve prompt section names → LoopPromptSection instances (fail-fast)
2. Construct AgentBudgetTracker with optional shared cap
3. Wire prompt sections into a `SectionAwareContentSummariser` decorator
4. Wire budget gate — `tryAcquire()` before each LLM invocation
5. Build emission policy from ObservationStrategy
6. Wire InFlightGate for concurrency control
7. Assemble SummarisationRunner
8. Construct and return AgentDecisionLoop

**Note on `outputBus`:** The factory receives `EventStreamBus<DecisionOutcome>`
as a constructor parameter. `DecisionLoopLevels.DECISION` is the output level
for decision loop outcomes.

**Note on `ContentSummariser` parameter:** The reasoner is passed in, not
constructed by the factory. This is intentional — the LLM invocation strategy
(which model, what prompt template, how to parse JSON) varies by use case. The
centralised coordinator will provide a routing-oriented summariser; a local
agent loop will provide a task-decision summariser. The factory wires the
plumbing (budget, sections, concurrency); the caller provides the intelligence.

**Note on `recordTokens()`:** Token recording is the reasoner implementation's
responsibility — it has access to the LLM response metadata. The budget tracker
is available via the loop's `budget()` accessor.

### SectionAwareContentSummariser

Decorator that calls `LoopPromptSection.contribute(context)` for each section
and makes the assembled prompt context available to the wrapped reasoner:

```java
class SectionAwareContentSummariser
        implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final ContentSummariser<ObservationEvent, DecisionOutcome> delegate;
    private final List<LoopPromptSection> sections;
    private final DecisionContext context;

    SectionAwareContentSummariser(
            ContentSummariser<ObservationEvent, DecisionOutcome> delegate,
            List<LoopPromptSection> sections,
            DecisionContext context) {
        this.delegate = delegate;
        this.sections = List.copyOf(sections);
        this.context = context;
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(
            List<ObservationEvent> items, @Nullable DecisionOutcome previous) {
        var promptContext = new StringBuilder();
        for (var section : sections) {
            var contribution = section.contribute(context);
            if (contribution != null && !contribution.isEmpty()) {
                promptContext.append(contribution).append("\n");
            }
        }
        // The assembled prompt context is available to the delegate via
        // a thread-local or by wrapping the events with the context.
        // Concrete wiring depends on the delegate's prompt assembly strategy.
        return delegate.summarise(items, previous);
    }
}
```

## AgentDecisionLoop

**Package:** `io.casehub.agency.orchestration.loop`
**Module:** agency-core

```java
public class AgentDecisionLoop {

    private static final System.Logger LOG =
        System.getLogger(AgentDecisionLoop.class.getName());

    private final DecisionLoopDescriptor descriptor;
    private final DecisionContext context;
    private final SummarisationRunner<ObservationEvent, DecisionOutcome> runner;
    private final AgentBudgetTracker budget;

    AgentDecisionLoop(DecisionLoopDescriptor descriptor,
                      DecisionContext context,
                      SummarisationRunner<ObservationEvent, DecisionOutcome> runner,
                      AgentBudgetTracker budget) {
        this.descriptor = descriptor;
        this.context = context;
        this.runner = runner;
        this.budget = budget;
    }

    public void collect(ObservationEvent event) {
        runner.collect(new LevelEvent<>(event, event.timestamp().toEpochMilli(),
            DecisionLoopLevels.OBSERVATION, null));
    }

    public CompletionStage<Void> tick(long now) {
        if (budget.isBudgetExhausted()) {
            LOG.log(System.Logger.Level.DEBUG,
                "Budget exhausted for {0} — skipping tick", context.agentId());
            return CompletableFuture.completedFuture(null);
        }
        return runner.tick(now);
    }

    public CompletionStage<Void> flush() {
        if (budget.isBudgetExhausted()) {
            return CompletableFuture.failedFuture(
                new BudgetExhaustedException(context.agentId()));
        }
        return runner.flush();
    }

    public void close() {
        runner.clear();
    }

    public DecisionLoopDescriptor descriptor() { return descriptor; }
    public DecisionContext context() { return context; }
    public AgentBudgetTracker budget() { return budget; }
}
```

### Scope filtering

Scope enforcement happens in the `ContentSummariser` implementation that
wraps the raw LLM output. The summariser:

1. Invokes the LLM with the assembled prompt
2. Parses the response into candidate `DecisionOutcome` variants
3. For each variant, determines its `DecisionType`
4. Checks `scope.isPermitted(decisionType)`
5. Permitted actions pass through; rejected actions become `ScopeGated`

This is the caller's responsibility when building the `ContentSummariser`
passed to the factory. The loop infrastructure (SummarisationRunner, emission
policy, budget) is agnostic to scope — it processes whatever the summariser
produces.

**Rationale:** Scope filtering is a property of the reasoning phase, not the
pipeline phase. The summariser knows what the LLM produced and what to filter.
The pipeline knows when to fire and how to accumulate — not what's permitted.

### DecisionType mapping conventions

Topology implementations that build the `ContentSummariser` must map LLM
output to `DecisionType` for scope filtering. Recommended mappings:

| DecisionOutcome variant | DecisionType | Rationale |
|-------------------------|-------------|-----------|
| WriteBack with signals only | SIGNALING | Signal deposit is communication |
| WriteBack with context writes | CONTEXT_MUTATION | Modifying shared state |
| WriteBack with observations | SIGNALING | Observations are passive signals |
| Route (Selected/SwarmAssignment) | TASK_SELECTION or COLLABORATION | Depends on whether selecting or coordinating |
| Aggregate | n/a | Aggregation is synthesis, not a decision that needs scope gating |

A single `WriteBack` can span multiple decision types — each sub-list should be
checked independently. The topology's summariser splits multi-typed WriteBacks
into separate scope checks.

### EventLevel constants

`EventLevel` is a record (`EventLevel(String name, int ordinal)`). Add constants
in a new holder class in the loop package:

```java
public final class DecisionLoopLevels {
    public static final EventLevel OBSERVATION = new EventLevel("OBSERVATION", 50);
    public static final EventLevel DECISION = new EventLevel("DECISION", 60);
    private DecisionLoopLevels() {}
}
```

Ordinals are above the existing summarisation levels, below platform event levels.

## Integration — Assembly Flow

How a topology implementation creates and uses a loop:

```java
// 1. Build descriptor (from YAML in #385, programmatic here)
var descriptor = new DecisionLoopDescriptor(
    "centralised-routing",
    ObservationStrategy.EVENT_DRIVEN,
    DecisionScope.FULL_AUTONOMY,
    CommunicationMode.STIGMERGY,
    AgentBudgetPolicy.defaults(),
    List.of("experience-replay", "role-context", "progress")
);

// 2. Build decision context for this agent
var context = new DecisionContext(
    caseId, agentId, descriptor.scope(),
    descriptor.communicationMode(), topology);

// 3. Provide the reasoning summariser
ContentSummariser<ObservationEvent, DecisionOutcome> reasoner = (events, prev) -> {
    // assemble prompt from sections + events
    // invoke LLM
    // parse response into DecisionOutcome
    // apply scope filter
    return CompletableFuture.completedFuture(outcome);
};

// 4. Create the loop
var loop = factory.create(descriptor, context, reasoner, sharedTracker);

// 5. Feed observations
loop.collect(ObservationEvent.of("signal.quality", 0.85));
loop.collect(ObservationEvent.of("context.plan.step", "step-3"));

// 6. Tick (emission policy decides whether to reason)
loop.tick(System.currentTimeMillis());

// 7. DecisionOutcome appears on the output bus → topology executes it
```

## What This Issue Does NOT Include

| Concern | Deferred to | Rationale |
|---------|-------------|-----------|
| YAML `.loop.yaml` parsing | agency#385 | YAML parity is a separate concern; Java types are YAML-aware |
| CoordinationTopology SPI | agency#380 | Concrete topology implementations that USE the loop |
| LlmSwarmCoordinator refactor | agency#381 | CentralisedTopology wraps a loop with routing semantics |
| HierarchicalInvocationTracker | agency#382 | Hierarchical budget management across loop trees |
| Per-agent degradation | agency#383 | Cascade policy when budget exhausts |
| DecisionLoopEvent observability | agency#384 | Event emission for loop lifecycle/decisions |
| Concrete ContentSummariser impls | agency#380/#381 | Routing-summariser, aggregation-summariser, local-decision-summariser |

## Testing Strategy

### 1. DecisionLoopDescriptor construction tests
- Valid construction with all parameters
- Null validation (requireNonNull for all required fields)
- Immutability — modifying source list doesn't affect descriptor
- `promptSectionNames` null → empty list

### 2. AgentBudgetPolicy tests
- Defaults factory method returns expected values
- Unlimited factory method returns MAX_VALUE
- Negative values throw

### 3. DecisionContext construction tests
- Valid construction (5 fields, no workState)
- Null validation for all required fields

### 4. DecisionOutcome variant tests
- Each variant constructs correctly
- Immutability on list fields (WriteBack.observations, Composite.outcomes)
- ScopeGated carries the attempted type and rejected outcome

### 5. AgentBudgetTracker tests
- Per-agent budget: tryAcquire succeeds up to limit, then fails
- Token tracking: recordTokens accumulates, isBudgetExhausted when exceeded
- Shared cap: tryAcquire fails when shared cap exhausted even if agent budget remains
- Shared cap: recordTokens propagates to shared tracker
- No shared cap: standalone operation

### 6. MapPromptSectionRegistry tests
- Resolves known names
- Throws for unknown names with helpful message
- resolveAll returns ordered list

### 7. DecisionLoopFactory tests
- Creates loop with EVENT_DRIVEN → WindowPolicy.ofCount(1)
- Creates loop with TICK_DRIVEN → WindowPolicy.ofDuration
- Creates loop with DEMAND_DRIVEN → never-emit policy
- Resolves prompt sections from registry
- Fails fast on unknown prompt section name
- Wires InFlightGate
- Wires budget gate — tryAcquire() called before summarisation
- Wires prompt sections into SectionAwareContentSummariser

### 8. AgentDecisionLoop integration tests
- collect() → tick() → DecisionOutcome appears on output bus (EVENT_DRIVEN)
- flush() forces processing regardless of emission policy (DEMAND_DRIVEN)
- Budget exhausted → tick() is no-op
- Budget exhausted → flush() returns failed future with BudgetExhaustedException
- Multiple observations accumulated → single summarisation call
- close() clears accumulated events

### 9. Scope filtering tests (in ContentSummariser wrapper)
- Permitted action passes through
- Unpermitted action becomes ScopeGated
- Composite with mixed permitted/unpermitted: permitted pass, unpermitted become ScopeGated

## Files — New

All in `agency-core/src/main/java/io/casehub/agency/orchestration/loop/`:

| File | Type | Purpose |
|------|------|---------|
| `DecisionLoopDescriptor.java` | record | Playbook descriptor |
| `AgentDecisionLoop.java` | class | Loop runtime |
| `DecisionLoopFactory.java` | class | Assembly from descriptor |
| `DecisionOutcome.java` | sealed interface | Unified action output |
| `DecisionContext.java` | record | Per-agent coordination view |
| `ObservationEvent.java` | record | Pipeline input type |
| `AgentBudgetPolicy.java` | record | Per-agent budget configuration |
| `AgentBudgetTracker.java` | class | Per-agent budget with shared cap |
| `BudgetExhaustedException.java` | class | Thrown by flush() when budget is spent |
| `PromptSectionRegistry.java` | interface | Named section resolution |
| `MapPromptSectionRegistry.java` | class | In-memory registry impl |
| `SectionAwareContentSummariser.java` | class | Prompt section decorator for ContentSummariser |
| `SignalDeposit.java` | record | WriteBack sub-record |
| `ContextWrite.java` | record | WriteBack sub-record |
| `LoopPromptSection.java` | interface | Loop-scoped prompt section (takes DecisionContext) |
| `DecisionLoopLevels.java` | class | EventLevel constants (OBSERVATION, DECISION) |

## Decisions

See `decisions-379.md` in this directory. Key choices:

- D1: ContentSummariser for LLM invocation (not StructuredAgentInvoker)
- D2: Sealed DecisionOutcome as unified action type
- D3: Compose with SummarisationRunner for trigger mechanism
- D4: Scope enforcement inside the loop (agency-side visibility)
- D5: Per-agent DecisionContext (renamed from ObservationContext; not shared SwarmContext)
- D6: Per-agent budget with swarm cap composite
- D7: Descriptor + runtime split (playbook model)
- D8: YAML-aware design, Java-first delivery
- D9: Prompt section references by name from registry
- D10: Loop produces DecisionOutcome, doesn't execute

## References

- `ContentSummariser.java` (summarisation-api) — LLM invocation primitive
- `SummarisationRunner.java` (summarisation-api) — tick-driven pipeline
- `EmissionPolicy.java` (summarisation-api) — emission control
- `ChangeGatedEmissionPolicy.java` (stigmergy/llm) — key-change trigger
- `InFlightGate.java` (stigmergy/llm) — concurrency guard
- `SwarmPromptSection.java` (swarm) — composable prompt fragment
- `LlmSwarmCoordinator.java` (swarm) — existing central coordinator (to be refactored in #381)
- `SwarmInvocationTracker.java` (swarm) — existing budget tracking
- `EnvironmentWriter.java` (stigmergy/llm) — write-back interface
- `RoutingDecision.java` (routing) — return-value action type
- `AggregationResult.java` (aggregation) — return-value action type
- `DecisionScope.java` (engine api) — scope enforcement
- `ObservationStrategy.java` (engine api) — trigger vocabulary
- `CommunicationMode.java` (engine api) — communication vocabulary
- `AgentWorkState.java` (engine api) — lifecycle state
- `TopologyDescriptor.java` (engine api) — topology snapshot
- `AgentWorkLifecycleListener.java` (engine api) — lifecycle events
- agency#377 — cross-repo parent epic
- agency#378 — agency sub-epic
- agency#379 — this issue
- agency#385 — YAML parity (future: `.loop.yaml` descriptor format)
- engine#1282 spec — coordination vocabulary types
- engine#1285 spec — agent work lifecycle design
