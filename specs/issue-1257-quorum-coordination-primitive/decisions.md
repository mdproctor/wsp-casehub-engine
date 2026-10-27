## D1: DSL placement — both `requires` guard and `quorumBarrier` trigger

**Choice:** Implement both mechanisms — `requires` as a generic binding guard and `quorumBarrier` as a new trigger type
**Alternatives:**
- `requires` only — covers all cases but forces bindings to carry a redundant trigger when the barrier IS the reason to fire
- `quorumBarrier` trigger only — clean for pipeline barriers but can't compose with other triggers (schedule + quorum, contextChange + quorum)
- Named groups + barrier — two-part declaration adds complexity without clear benefit over direct binding references
**Rationale:** `requires` is the generic mechanism that composes with any trigger. `quorumBarrier` is syntactic sugar for the common pipeline case where the barrier IS the trigger. Both are needed: aggregators in pipelines use `quorumBarrier`; bindings that fire on schedule/context-change but also need a quorum gate use `requires`.
**Trade-offs:** Two mechanisms for related concepts. Mitigated by the fact that `quorumBarrier` is semantically a trigger (like scopeActivated) while `requires` is semantically a guard (like `when`).
**Sources:** CaseDefinition.yaml Trigger type (lines 814-836), Binding.when guard (line 122-124), PlanningStrategyLoopControl dispatch pipeline
**Exploration:** quick
**Status:** captured

## D2: Threshold semantics — COMPLETED only

**Choice:** Only count COMPLETED bindings toward the quorum threshold
**Alternatives:**
- Any terminal state — COMPLETED + FAULTED + REJECTED all count. More flexible but harder to reason about data quality.
- Configurable per barrier — `countTerminal: true` flag. Adds DSL complexity for a rare use case.
**Rationale:** Clean semantics — the aggregator knows it has good data from N sources. Fault tolerance is handled by existing mechanisms (contingency bindings, outcomePolicy, recovery).
**Trade-offs:** A pipeline where "any 2 finished, regardless of outcome" is valid cannot express that. Rare pattern — file a follow-up if it surfaces.
**Sources:** CompletionSemantics.MOfN evaluation in DefaultCasePlanModel.evaluateCompletion(), SubCaseGroupPolicy threshold logic
**Exploration:** quick
**Status:** captured

## D3: Straggler policy — KEEP

**Choice:** Let remaining bindings run to completion after quorum threshold is met (default KEEP)
**Alternatives:**
- CANCEL — cancel remaining bindings once threshold met. Saves resources but loses potential data.
- Configurable via OnThresholdReached — adds DSL complexity.
**Rationale:** Consistent with SubCase's default. Most multi-agent scenarios benefit from additional data merging into context. CANCEL can be added later as an optional `onThresholdReached` property if needed.
**Trade-offs:** Cannot short-circuit expensive workers. Acceptable for v1 — CANCEL is additive.
**Sources:** OnThresholdReached enum (api module), SubCase.onThresholdReached in CaseDefinition.yaml (line 781-784)
**Exploration:** quick
**Depends on:** D2 (threshold semantics determine what "remaining" means)
**Status:** captured

## D4: Scope — include compound MofN YAML wiring fix

**Choice:** Fix compound-level MofN YAML wiring within this issue
**Alternatives:**
- Separate issue — cleaner scope but leaves a silent bug while we're already working in the same area
**Rationale:** Small delta. `mofnRequired` is dropped during `YamlCaseDefinitionConverter.convertCompound()` and no `BlackboardPlanConfigurer` bridges YAML compounds to runtime. Fixing this while adding the binding-level quorum keeps the "M-of-N coordination" story coherent. Both are about the same runtime machinery (`CompletionSemantics`, `PlanningStrategyLoopControl`).
**Trade-offs:** Slightly larger PR. Mitigated by the fact that both changes are in the same modules.
**Sources:** YamlCaseDefinitionConverter.convertCompound() (line 1315), BlackboardPlanConfigurer SPI, DefaultCasePlanModel
**Exploration:** quick
**Status:** captured

## D5: Runtime hook — pre-strategy filter in PlanningStrategyLoopControl

**Choice:** Evaluate quorum barriers in PlanningStrategyLoopControl after compound gating but before strategy dispatch
**Alternatives:**
- Inside PlanningStrategy — each strategy evaluates barriers. More flexible but every strategy must handle it.
- New LoopControl decorator — clean separation but adds an abstraction layer for a single concern.
**Rationale:** PlanningStrategyLoopControl already gates bindings by compound status. Adding quorum barrier evaluation at the same level keeps the dispatch pipeline linear: compound gating → quorum gating → strategy dispatch. No strategy changes needed.
**Trade-offs:** Strategies cannot factor quorum into priority decisions. Acceptable — quorum is a hard gate, not a soft preference.
**Sources:** PlanningStrategyLoopControl.select() dispatch pipeline, CompoundStrategyDispatcher
**Exploration:** quick
**Depends on:** D1 (requires vs quorumBarrier determines what the filter checks)
**Status:** captured
