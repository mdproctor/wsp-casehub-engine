# Goal Formation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #805 — Goal formation — agents discover new goals from experience
**Issue group:** #803, #806, #805, #808

**Goal:** Enable agents to discover and register new goals from reflection
insights and accumulated memories, completing the goal lifecycle pipeline.

**Architecture:** GoalFormationEvaluator is called from AgentExperienceRecorder
after ReflectionOrchestrator.reflect() returns insights. It retrieves recent
memories, passes insights + goals + memories to GoalFormationStrategy (LLM),
validates proposals against AgentGoal constraints, and registers via
AgentRegistry with a config-based approval gate.

**Tech Stack:** Java 21, Quarkus 3.32, eidos-api (AgentRegistry, AgentGoal,
AgentDescriptor), neocortex-memory-api (CaseMemoryStore, Memory, MemoryQuery),
Mutiny Uni, virtual threads

## Global Constraints

- Pre-release platform — breaking changes are free
- IntelliJ MCP required for all code navigation and editing
- TDD: failing test first, then implementation
- Every commit references an issue: `Refs #805`
- `Instance<>` injection with `isResolvable()` guard for optional dependencies
- `@ApplicationScoped` for all CDI beans
- `NamedStrategy` convention for strategy SPI (id(), EngineStrategyResolver)

---

### Task 1: Engine-api SPI types + event types

Create GoalFormationStrategy SPI, GoalFormationContext, GoalFormationProposal,
and GOAL_FORMED/GOAL_PROPOSED event type constants.

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalFormationStrategy.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalFormationContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/GoalFormationProposal.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/GoalFormationProposalTest.java`

**Interfaces:**
- Consumes: `NamedStrategy` (platform-api), `AgentGoal` (eidos-api), `GoalPriority` (eidos-api), `RetrievedMemory` (engine-api), `CaseDefinition` (engine-api)
- Produces: `GoalFormationStrategy` (used by Tasks 3, 4), `GoalFormationContext` (used by Tasks 3, 4), `GoalFormationProposal` (used by Tasks 3, 4), `GOAL_FORMED`/`GOAL_PROPOSED` event types (used by Task 3)

- [ ] **Step 1: Create GoalFormationProposal record**

```java
package io.casehub.api.spi.routing;

import io.casehub.eidos.api.GoalPriority;
import java.util.List;
import java.util.Objects;

public record GoalFormationProposal(
    List<ProposedGoal> goals,
    String rationale
) {
    public GoalFormationProposal {
        Objects.requireNonNull(goals, "goals must not be null");
        goals = List.copyOf(goals);
    }

    public record ProposedGoal(
        String name,
        String description,
        GoalPriority suggestedPriority,
        String formationReason
    ) {
        public ProposedGoal {
            Objects.requireNonNull(name, "name must not be null");
            Objects.requireNonNull(description, "description must not be null");
            Objects.requireNonNull(formationReason, "formationReason must not be null");
        }
    }
}
```

- [ ] **Step 2: Create GoalFormationContext record**

```java
package io.casehub.api.spi.routing;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.RetrievedMemory;
import io.casehub.eidos.api.AgentGoal;
import java.util.List;
import java.util.Objects;

public record GoalFormationContext(
    String agentId,
    String tenancyId,
    List<String> reflectionInsights,
    List<AgentGoal> existingGoals,
    List<RetrievedMemory> recentMemories,
    int remainingCapacity,
    CaseDefinition definition
) {
    public GoalFormationContext {
        Objects.requireNonNull(agentId, "agentId must not be null");
        Objects.requireNonNull(tenancyId, "tenancyId must not be null");
        Objects.requireNonNull(reflectionInsights, "reflectionInsights must not be null");
        reflectionInsights = List.copyOf(reflectionInsights);
        Objects.requireNonNull(existingGoals, "existingGoals must not be null");
        existingGoals = List.copyOf(existingGoals);
        Objects.requireNonNull(recentMemories, "recentMemories must not be null");
        recentMemories = List.copyOf(recentMemories);
    }
}
```

- [ ] **Step 3: Create GoalFormationStrategy interface**

```java
package io.casehub.api.spi.routing;

import io.casehub.platform.api.routing.NamedStrategy;
import io.smallrye.mutiny.Uni;

public interface GoalFormationStrategy extends NamedStrategy {
    Uni<GoalFormationProposal> propose(GoalFormationContext context);

    @Override
    default String id() { return "llm"; }
}
```

- [ ] **Step 4: Add GOAL_FORMED and GOAL_PROPOSED to CaseHubEventType**

Use `ide_edit_member` to add after `GOAL_REVISED`:

```java
GOAL_FORMED,
GOAL_PROPOSED,
```

- [ ] **Step 5: Write GoalFormationProposalTest**

```java
package io.casehub.api.spi.routing;

import io.casehub.eidos.api.GoalPriority;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class GoalFormationProposalTest {

    @Test
    void constructsWithValidData() {
        var goal = new GoalFormationProposal.ProposedGoal(
            "new-goal", "A new goal", GoalPriority.SECONDARY, "emerged from experience");
        var proposal = new GoalFormationProposal(List.of(goal), "test rationale");
        assertEquals(1, proposal.goals().size());
        assertEquals("new-goal", proposal.goals().get(0).name());
    }

    @Test
    void goalsListIsImmutable() {
        var proposal = new GoalFormationProposal(List.of(), "rationale");
        assertThrows(UnsupportedOperationException.class,
            () -> proposal.goals().add(
                new GoalFormationProposal.ProposedGoal(
                    "g", "d", GoalPriority.SECONDARY, "r")));
    }

    @Test
    void nullGoalsThrows() {
        assertThrows(NullPointerException.class,
            () -> new GoalFormationProposal(null, "rationale"));
    }

    @Test
    void nullNameThrows() {
        assertThrows(NullPointerException.class,
            () -> new GoalFormationProposal.ProposedGoal(
                null, "desc", GoalPriority.SECONDARY, "reason"));
    }

    @Test
    void nullPriorityAllowed() {
        var goal = new GoalFormationProposal.ProposedGoal(
            "g", "d", null, "reason");
        assertNull(goal.suggestedPriority());
    }

    @Test
    void nullFormationReasonThrows() {
        assertThrows(NullPointerException.class,
            () -> new GoalFormationProposal.ProposedGoal(
                "g", "d", GoalPriority.SECONDARY, null));
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl api -Dtest=GoalFormationProposalTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/routing/GoalFormationStrategy.java api/src/main/java/io/casehub/api/spi/routing/GoalFormationContext.java api/src/main/java/io/casehub/api/spi/routing/GoalFormationProposal.java api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java api/src/test/java/io/casehub/api/spi/routing/GoalFormationProposalTest.java
git commit -m "feat(#805): add GoalFormationStrategy SPI types + GOAL_FORMED/GOAL_PROPOSED event types"
```

---

### Task 2: GoalFormationEvaluator

Core evaluator with cooldown, capacity check, memory retrieval,
strategy delegation, validation, and config-based approval gate.

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/GoalFormationEvaluator.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalFormationEvaluatorTest.java`

**Interfaces:**
- Consumes: `GoalFormationStrategy` (Task 1), `GoalFormationContext` (Task 1), `GoalFormationProposal` (Task 1), `AgentRegistry` (eidos-api), `CaseMemoryStore` (neocortex), `CaseDefinitionRegistry` (engine-common), `EngineStrategyResolver` (runtime), `EventLogRepository` (engine-common), `GOAL_FORMED`/`GOAL_PROPOSED` (Task 1)
- Produces: `GoalFormationEvaluator.evaluate(String workerName, CaseInstance caseInstance, List<String> insights)` — called by Task 4 (handler wiring)

- [ ] **Step 1: Write test — skips when not enabled**

```java
@Test
void skipsWhenNotEnabled() {
    var disabled = buildEvaluator(false, true, "llm", 2, 60, 20);
    disabled.evaluate("worker-1", buildCaseInstance("tenant-1"), List.of("insight"));
    verify(agentRegistry, never()).findById(any(), any());
}
```

- [ ] **Step 2: Write test — skips when AgentRegistry not resolvable**

- [ ] **Step 3: Write test — skips when no descriptor**

- [ ] **Step 4: Write test — skips during cooldown**

```java
@Test
void skipsDuringCooldown() throws Exception {
    setupDefinitionWithGoals(instance, "worker-1", goal("g1"));
    when(agentRegistry.findById("agent-1", "tenant-1"))
        .thenReturn(Optional.of(descriptorWithGoals(goal("g1"))));

    GoalFormationStrategy strategy = mock(GoalFormationStrategy.class);
    when(strategy.propose(any())).thenReturn(
        Uni.createFrom().item(new GoalFormationProposal(List.of(), "none")));
    when(strategyResolver.resolve(GoalFormationStrategy.class, "llm")).thenReturn(strategy);

    evaluator.evaluate("worker-1", instance, List.of("insight"));
    Thread.sleep(100);
    evaluator.evaluate("worker-1", instance, List.of("insight-2"));
    Thread.sleep(200);
    verify(strategy, times(1)).propose(any());
}
```

- [ ] **Step 5: Write test — skips when capacity full (10 existing goals)**

- [ ] **Step 6: Write test — skips when insights empty**

- [ ] **Step 7: Write test — calls strategy with correct context**

Verify that `GoalFormationContext` passed to `strategy.propose()` contains
the reflection insights, existing goals, retrieved memories, and correct
remaining capacity.

- [ ] **Step 8: Write test — validates proposed goals and rejects invalid**

```java
@Test
void rejectsGoalWithNameTooLong() throws Exception {
    // ... setup
    var longName = "x".repeat(101);
    var proposal = new GoalFormationProposal(
        List.of(new GoalFormationProposal.ProposedGoal(
            longName, "desc", GoalPriority.SECONDARY, "reason")),
        "rationale");
    when(strategy.propose(any())).thenReturn(Uni.createFrom().item(proposal));

    evaluator.evaluate("worker-1", instance, List.of("insight"));
    Thread.sleep(300);
    verify(agentRegistry, never()).register(any());
}
```

- [ ] **Step 9: Write test — rejects duplicate goal names**

- [ ] **Step 10: Write test — caps at maxNewGoalsPerReflection**

- [ ] **Step 11: Write test — auto-approve=true registers via AgentRegistry**

```java
@Test
void autoApproveRegistersNewGoals() throws Exception {
    setupDefinitionWithGoals(instance, "worker-1", goal("existing-goal"));
    when(agentRegistry.findById("agent-1", "tenant-1"))
        .thenReturn(Optional.of(descriptorWithGoals(goal("existing-goal"))));

    var proposed = new GoalFormationProposal.ProposedGoal(
        "new-goal", "A new goal", GoalPriority.SECONDARY, "from insight");
    when(strategy.propose(any())).thenReturn(
        Uni.createFrom().item(new GoalFormationProposal(List.of(proposed), "rationale")));

    CountDownLatch latch = new CountDownLatch(1);
    doAnswer(inv -> { registeredDescriptors.add(inv.getArgument(0)); latch.countDown(); return null; })
        .when(agentRegistry).register(any());

    evaluator.evaluate("worker-1", instance, List.of("insight"));
    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();

    AgentDescriptor registered = registeredDescriptors.get(0);
    assertThat(registered.goals()).hasSize(2);
    assertThat(registered.goals().stream().map(AgentGoal::name).toList())
        .containsExactlyInAnyOrder("existing-goal", "new-goal");
}
```

- [ ] **Step 12: Write test — auto-approve=false writes GOAL_PROPOSED only**

- [ ] **Step 13: Write test — defaults suggestedPriority to SECONDARY when null**

- [ ] **Step 14: Write test — per-goal error isolation**

- [ ] **Step 15: Write test — exception isolation — never blocks**

- [ ] **Step 16: Write test — writes GOAL_FORMED EventLog with metadata**

- [ ] **Step 17: Implement GoalFormationEvaluator**

```java
@ApplicationScoped
public class GoalFormationEvaluator {

  private static final Logger LOG = Logger.getLogger(GoalFormationEvaluator.class);
  private static final int MAX_GOALS = 10;

  private final Instance<AgentRegistry> agentRegistry;
  private final Instance<CaseMemoryStore> caseMemoryStore;
  private final CaseDefinitionRegistry caseDefinitionRegistry;
  private final EngineStrategyResolver strategyResolver;
  private final EventLogRepository eventLogRepository;
  private final boolean enabled;
  private final boolean autoApprove;
  private final String strategyId;
  private final int maxNewPerReflection;
  private final long cooldownMinutes;
  private final int maxMemories;

  private final ConcurrentHashMap<String, Instant> lastFormationTime = new ConcurrentHashMap<>();

  @Inject
  public GoalFormationEvaluator(
      Instance<AgentRegistry> agentRegistry,
      Instance<CaseMemoryStore> caseMemoryStore,
      CaseDefinitionRegistry caseDefinitionRegistry,
      EngineStrategyResolver strategyResolver,
      EventLogRepository eventLogRepository,
      @ConfigProperty(name = "casehub.engine.goal.formation.enabled", defaultValue = "false")
          boolean enabled,
      @ConfigProperty(name = "casehub.engine.goal.formation.auto-approve", defaultValue = "true")
          boolean autoApprove,
      @ConfigProperty(name = "casehub.engine.goal.formation.strategy", defaultValue = "llm")
          String strategyId,
      @ConfigProperty(name = "casehub.engine.goal.formation.max-new-per-reflection", defaultValue = "2")
          int maxNewPerReflection,
      @ConfigProperty(name = "casehub.engine.goal.formation.cooldown-minutes", defaultValue = "60")
          long cooldownMinutes,
      @ConfigProperty(name = "casehub.engine.goal.formation.max-memories", defaultValue = "20")
          int maxMemories) {
    // assign all fields
  }

  public void evaluate(String workerName, CaseInstance caseInstance, List<String> insights) {
    if (!enabled) return;
    if (!agentRegistry.isResolvable()) return;
    if (insights == null || insights.isEmpty()) return;

    CaseDefinition definition;
    try {
      definition = caseDefinitionRegistry.getCaseDefinition(caseInstance.getCaseMetaModel());
    } catch (Exception e) { return; }

    Optional<AgentDescriptor> descOpt = definition.agentDescriptorFor(workerName);
    if (descOpt.isEmpty()) return;

    AgentDescriptor descriptor = descOpt.get();
    String agentId = descriptor.agentId();
    String tenancyId = caseInstance.tenancyId;
    String key = agentId + "|" + tenancyId;

    // cooldown check
    Instant last = lastFormationTime.get(key);
    if (last != null && Duration.between(last, Instant.now()).toMinutes() < cooldownMinutes) return;

    int remaining = MAX_GOALS - descriptor.goals().size();
    if (remaining <= 0) return;

    // retrieve memories
    List<RetrievedMemory> memories = retrieveMemories(workerName, tenancyId);

    // resolve strategy
    GoalFormationStrategy strategy;
    try {
      strategy = strategyResolver.resolve(GoalFormationStrategy.class, strategyId);
    } catch (Exception e) { return; }

    try {
      GoalFormationContext context = new GoalFormationContext(
          agentId, tenancyId, insights, descriptor.goals(), memories, remaining, definition);
      GoalFormationProposal proposal = strategy.propose(context).await().indefinitely();
      if (proposal == null || proposal.goals().isEmpty()) return;

      List<AgentGoal> newGoals = validateAndConvert(proposal.goals(), descriptor, remaining);
      if (newGoals.isEmpty()) return;

      if (autoApprove) {
        List<AgentGoal> merged = new ArrayList<>(descriptor.goals());
        merged.addAll(newGoals);
        AgentDescriptor updated = descriptor.toBuilder().goals(merged).build();
        agentRegistry.get().register(updated);
        writeAuditLog(agentId, tenancyId, newGoals, descriptor.goals().size(),
            merged.size(), insights.size(), memories.size(), CaseHubEventType.GOAL_FORMED);
      } else {
        writeAuditLog(agentId, tenancyId, newGoals, descriptor.goals().size(),
            descriptor.goals().size(), insights.size(), memories.size(), CaseHubEventType.GOAL_PROPOSED);
      }

      lastFormationTime.put(key, Instant.now());
    } catch (Exception e) {
      LOG.warnf(e, "Goal formation failed for agent %s", agentId);
    }
  }

  // validateAndConvert, retrieveMemories, writeAuditLog methods
}
```

- [ ] **Step 18: Run GoalFormationEvaluator tests**

Run: `mvn test -pl runtime -Dtest=GoalFormationEvaluatorTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: ALL PASS

- [ ] **Step 19: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/routing/GoalFormationEvaluator.java runtime/src/test/java/io/casehub/engine/internal/routing/GoalFormationEvaluatorTest.java
git commit -m "feat(#805): add GoalFormationEvaluator with cooldown, validation, and approval gate"
```

---

### Task 3: LlmGoalFormationStrategy

LLM-backed GoalFormationStrategy implementation.

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/LlmGoalFormationStrategy.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/LlmGoalFormationStrategyTest.java`

**Interfaces:**
- Consumes: `GoalFormationStrategy` (Task 1), `GoalFormationContext` (Task 1), `GoalFormationProposal` (Task 1), `ChatModelProvider` (engine-api), `Agent` (engine-api)
- Produces: `LlmGoalFormationStrategy` — resolved by `EngineStrategyResolver` via id=`"llm"`

- [ ] **Step 1: Write test — produces proposal from canned LLM response**

```java
@Test
void producesProposalFromLlmResponse() {
    ChatModel mockModel = input -> WorkerResult.of(Map.of(
        "goals", List.of(Map.of(
            "name", "optimize-reviews",
            "description", "Optimize code review turnaround",
            "suggestedPriority", "SECONDARY",
            "formationReason", "Pattern of delayed reviews observed")),
        "rationale", "Reflection insights indicate review bottleneck"));

    when(chatModelProviders.isUnsatisfied()).thenReturn(false);
    when(chatModelProviders.get()).thenReturn(() -> mockModel);

    GoalFormationContext context = new GoalFormationContext(
        "agent-1", "tenant-1",
        List.of("Review turnaround has been slow"),
        List.of(goal("find-bugs")),
        List.of(), 9, null);

    GoalFormationProposal proposal = strategy.propose(context).await().indefinitely();
    assertThat(proposal.goals()).hasSize(1);
    assertThat(proposal.goals().get(0).name()).isEqualTo("optimize-reviews");
}
```

- [ ] **Step 2: Write test — no-op when ChatModelProvider absent**

- [ ] **Step 3: Write test — empty goals array is valid**

- [ ] **Step 4: Write test — includes memories in prompt**

- [ ] **Step 5: Implement LlmGoalFormationStrategy**

Follow the exact pattern from `LlmGoalRevisionStrategy`: inject
`Instance<ChatModelProvider>`, build `Agent` with system prompt, call
`agent.execute()`, parse response JSON.

System prompt: "You are a goal discovery analyst. Given an agent's recent
reflection insights, its current goals, and relevant memories, identify
new goals the agent should pursue. Only propose goals that represent
genuinely new objectives — not refinements of existing goals. Each goal
must be specific, actionable, and distinct from existing goals. Respond
with JSON only."

User prompt includes: reflection insights, existing goals with priorities,
recent memories (text), remaining capacity.

Response schema:
```json
{"goals": [{"name": "...", "description": "...", "suggestedPriority": "SECONDARY"|null, "formationReason": "..."}], "rationale": "..."}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl runtime -Dtest=LlmGoalFormationStrategyTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/routing/LlmGoalFormationStrategy.java runtime/src/test/java/io/casehub/engine/internal/routing/LlmGoalFormationStrategyTest.java
git commit -m "feat(#805): add LlmGoalFormationStrategy"
```

---

### Task 4: Wire GoalFormationEvaluator into AgentExperienceRecorder

Add GoalFormationEvaluator call after reflect() returns insights.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/memory/AgentExperienceRecorder.java`

**Interfaces:**
- Consumes: `GoalFormationEvaluator.evaluate()` (from Task 2)
- Produces: Complete wiring — goal formation fires after every reflection

- [ ] **Step 1: Inject GoalFormationEvaluator into AgentExperienceRecorder**

Add field: `private final GoalFormationEvaluator goalFormationEvaluator;`
Add constructor parameter.

- [ ] **Step 2: Add call after reflect() in evaluateReflectionTrigger**

In the virtual thread lambda (line 138-148), after `reflect()` returns,
call the evaluator:

```java
Thread.startVirtualThread(
    () -> {
      try {
        List<String> insights = reflectionOrchestrator
            .get()
            .reflect(workerName, caseInstance.tenancyId, sinceFinal, config.maxSourceMemories());
        goalFormationEvaluator.evaluate(workerName, caseInstance, insights);
      } catch (Exception e) {
        LOG.warnf(e, "Reflection failed for agent %s", workerName);
      }
    });
```

Note: `reflect()` currently returns void in some contexts but `List<String>`
per the `ReflectionOrchestrator` interface. Capture the return value and
pass to the evaluator.

- [ ] **Step 3: Run compilation check**

Run: `mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/memory/AgentExperienceRecorder.java
git commit -m "feat(#805): wire GoalFormationEvaluator into AgentExperienceRecorder"
```

---

### Task 5: Integration test + CLAUDE.md

Full-flow `@QuarkusTest` verifying the goal formation pipeline, and
CLAUDE.md documentation update.

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/GoalFormationIntegrationTest.java`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: All components from Tasks 1-4

- [ ] **Step 1: Write integration test**

Follow the pattern from `GoalRevisionIntegrationTest`:
- `@QuarkusTest @TestProfile` with config overrides enabling goal formation
  and setting `cooldown-minutes=0` for test speed
- Inner `TestGoalSignalStore`, `TestGoalEvolution`, `TestAgentRegistry`
  (reuse the pattern from GoalRevisionIntegrationTest)
- Inner `GoalFormationCaseHub extends CaseHub` with an agent descriptor
  that has one existing goal
- Mock `ReflectionOrchestrator` that returns canned insights
- Mock `ChatModelProvider` returning canned goal proposal JSON
- Start enough cases to trigger reflection threshold
- Verify new goal appears in AgentRegistry
- Verify GOAL_FORMED EventLog entry

Test structure:
1. Seed AgentRegistry with initial descriptor (1 goal)
2. Provide mock ReflectionOrchestrator returning `["Agent consistently improves code quality"]`
3. Provide mock ChatModelProvider returning `{goals: [{name: "quality-metrics", ...}], rationale: "..."}`
4. Start 3 cases (triggers reflection via AgentExperienceRecorder threshold)
5. Await AgentRegistry.registrations() containing descriptor with 2 goals
6. Verify the new goal has name "quality-metrics" and priority SECONDARY

- [ ] **Step 2: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=GoalFormationIntegrationTest -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: PASS

- [ ] **Step 3: Run full runtime test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -f /Users/mdproctor/claude/casehub/slots/94/engine/pom.xml`
Expected: ALL PASS (no regressions)

- [ ] **Step 4: Add Goal Formation section to CLAUDE.md**

Document GoalFormationEvaluator, GoalFormationStrategy SPI,
GoalFormationContext, LlmGoalFormationStrategy, configuration properties,
the AgentExperienceRecorder integration, and the approval gate mechanism.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/internal/routing/GoalFormationIntegrationTest.java CLAUDE.md
git commit -m "feat(#805): add GoalFormationIntegrationTest + CLAUDE.md documentation

Refs #805, Refs #808"
```
