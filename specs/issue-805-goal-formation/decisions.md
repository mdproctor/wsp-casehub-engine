# Decisions — #805 Goal Formation

## D1: Scope — merge #805 and #808

**Choice:** Full pipeline — merge #805 (mechanism) and #808 (trigger from memory) into a single design
**Alternatives:**
- Mechanism only (#805) — define SPI + evaluator; #808 wires reflection trigger separately. Clean separation but artificial split.
- Mechanism + basic trigger — #805 defines mechanism + simple threshold trigger; #808 adds memory-based discovery. Middle ground but still two specs.
**Rationale:** The trigger and mechanism are tightly coupled — reflection insights are the formation context. Designing them separately risks interface mismatches.
**Trade-offs:** Larger single design; #808 becomes a sub-task rather than its own spec
**Exploration:** quick
**Status:** captured

## D2: Approval gate — config-based

**Choice:** Config-based gate (`casehub.engine.goal.formation.auto-approve`)
**Alternatives:**
- Auto-register — no gate, register immediately after validation. Too risky for first iteration.
- ActionGate integration — route through WorkItem approval. Wrong abstraction (case-level vs agent-level).
**Rationale:** `true` for dev/testing, `false` for production. When false, goals are logged as GOAL_PROPOSED but not registered. Fully testable without approval UX commitment.
**Trade-offs:** No built-in approval workflow in v1; external systems must observe GOAL_PROPOSED events
**Exploration:** quick
**Status:** captured

## D3: Trigger point — post-reflection in AgentExperienceRecorder

**Choice:** Call GoalFormationEvaluator from AgentExperienceRecorder after reflect() returns, on the same virtual thread
**Alternatives:**
- CDI event observer — ReflectionCompleted event + observer. Decoupled but unnecessary plumbing.
- Separate threshold trigger — independent accumulation, doesn't use reflection output. Misses the insight-to-goal connection.
**Rationale:** Same pattern as existing evaluator-per-completion components. Reflection insights are the direct input to formation.
**Trade-offs:** Coupled to AgentExperienceRecorder's virtual thread lifecycle
**Exploration:** quick
**Status:** captured

## D4: LLM context — insights + existing goals + recent memories

**Choice:** Pass reflection insights, current AgentDescriptor goals, and recent agent memories to the LLM
**Alternatives:**
- Insights + goals only — prevents duplicates but may miss richer patterns
- Insights only — minimal context, higher duplicate/irrelevant risk
**Rationale:** Richer context produces more relevant goals. Memory retrieval is already available via CaseMemoryStore.
**Trade-offs:** Heavier pipeline (memory retrieval adds latency); requires CaseMemoryStore availability
**Exploration:** quick
**Status:** captured

## D5: SPI design — dedicated GoalFormationStrategy

**Choice:** New GoalFormationStrategy extends NamedStrategy with propose(GoalFormationContext) → Uni<GoalFormationProposal>
**Alternatives:**
- Extend GoalRevisionStrategy — conflates revision and formation
- Generic GoalLifecycleStrategy — covers revision + formation + abandonment. Complex, hard to test independently.
**Rationale:** Formation and revision are different concerns (creating new vs modifying existing). Parallel SPI structure follows the established pattern.
**Trade-offs:** Another strategy interface to maintain
**Exploration:** quick
**Status:** captured

## D6: Rate limiting — per-reflection cap + cooldown

**Choice:** Max N new goals per reflection cycle (configurable, default 2) + cooldown period (configurable, default 1 hour)
**Alternatives:**
- Budget-based — per-agent attempts per time window. More flexible but complex state.
- Capacity-only — just check 10 - existing_count. No temporal pacing.
**Rationale:** Simple, predictable. Prevents both per-cycle flooding and sustained high-frequency formation. Capacity check (10 - existing) is always enforced on top.
**Trade-offs:** Fixed cap may miss valid multi-goal formation opportunities
**Exploration:** quick
**Status:** captured

## D7: Architecture — inline evaluator

**Choice:** GoalFormationEvaluator called inline from AgentExperienceRecorder, same virtual thread as reflection
**Alternatives:**
- Event-driven pipeline — ReflectionCompleted CDI event + observer. Unnecessary plumbing.
- Staged pipeline with persistence — GoalCandidateStore for async review. v2 territory.
**Rationale:** Follows established evaluator pattern (GoalRevisionEvaluator, PersonalitySignalRecorder). Proven, simple, tested.
**Trade-offs:** Extends virtual thread lifetime; no intermediate persistence for crash recovery
**Exploration:** quick
**Status:** captured
