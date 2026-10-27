## D1: DSL placement — `requires` guard only

**Choice:** Implement `requires` as a binding-level guard that evaluates against plan model state (binding completion status). Drop the `quorumBarrier` trigger — the `requires` guard is the sole quorum mechanism.
**Alternatives:**
- `requires` guard + `quorumBarrier` trigger — two mechanisms add cognitive load for related concepts. The pipeline case (quorum IS the activation) is handled by combining `requires` with `ScopeActivatedTrigger` in compound scope.
- `quorumBarrier` trigger only — violates Trigger sealed interface semantics: all four existing variants (`ContextChangeTrigger`, `ScheduleTrigger`, `ScopeActivatedTrigger`, `CloudEventTrigger`) respond to external events. A quorum barrier responds to internal plan model state transitions. While `ScopeActivatedTrigger` also responds to internal state (compound scope activation), adding a quorum trigger conflates two DSL dimensions (trigger vs. guard) that the platform keeps separate.
- Extend `when` guard — `when` evaluates against CaseContext via `ExpressionEngineRegistry.evaluate(ExpressionEvaluator, CaseContext)`. Plan model state (binding completion status) is not accessible through this path. `requires` is mechanically distinct: it evaluates against `CasePlanModel`, not `CaseContext`.
- Named groups + barrier — two-part declaration adds complexity without clear benefit over direct binding references.
**Rationale:** `requires` is a prerequisite guard that composes naturally with any trigger. For the pipeline case, the aggregator binding uses `ScopeActivatedTrigger` (fires when compound scope activates) combined with `requires` (holds dispatch until prerequisites are met). The runtime re-evaluates `requires`-gated bindings when referenced bindings complete — this is a dispatch-pipeline concern handled in `PlanningStrategyLoopControl.select()`. Composition ordering is natural: `requires` (plan model prerequisites) → `when` (case context guard) → trigger evaluation.
**Trade-offs:** Bindings whose sole reason to fire is "quorum met" must still carry a trigger (typically `ScopeActivatedTrigger`). This is consistent with the platform pattern — all bindings have triggers, guards are additional filters.
**Sources:** Trigger sealed interface (`api/Trigger.java`), PlanningStrategyLoopControl.evaluateWhenGuard() evaluates against ctx.caseContext(), ExpressionEngineRegistry.evaluate(ExpressionEvaluator, CaseContext) signature
**Exploration:** quick → revised under review
**Status:** revised (R1-02, R1-05: dropped quorumBarrier trigger, requires-only design)

## D2: Threshold semantics — COMPLETED only with unreachable-quorum detection

**Choice:** Count only COMPLETED bindings toward the quorum threshold. Additionally, detect unreachable quorum: when `(total - faulted) < required`, mark the barrier as FAILED immediately.
**Alternatives:**
- Any terminal state (`isTerminal()`) — counts COMPLETED + FAULTED + REJECTED + OBSOLETE + CANCELLED. This is what compound M-of-N uses (`DefaultCasePlanModel.evaluateCompletion()` line 298). More permissive but the aggregator receives results from faulted agents — bad data quality.
- COMPLETED-only without failure detection — the original D2 choice. Unreachable quorum blocks forever. SubCaseGroupPolicy already solves this: `remaining < needed → REJECTED`.
- Configurable per barrier — `countTerminal: true` flag. Adds DSL complexity for a rare use case.
**Rationale:** The SubCaseGroupPolicy pattern is the right model: threshold based on COMPLETED (data quality guarantee — the aggregator knows it has good data from N sources), plus early failure detection when the quorum becomes unreachable. This matches SubCase semantics: `group.getCompletedCount() >= group.getRequiredCount()` for threshold, `remaining < needed → REJECTED` for failure. The existing platform inconsistency (compound uses `isTerminal()`, SubCase uses `completedCount`) is intentional per-level design: compound completion asks "are children done?" (any terminal state counts), while quorum coordination asks "do we have enough good results?" (only successful completions count).
**Trade-offs:** A pipeline where "any 2 finished, regardless of outcome" is valid cannot express that directly. This is a rare pattern — if it surfaces, adding a `countTerminal: true` flag is additive.
**Sources:** DefaultCasePlanModel.evaluateCompletion() line 298 uses isTerminal(), SubCaseGroupPolicy.evaluate() uses completedCount with remaining < needed → REJECTED, CompoundPlanModelTest lines 191 and 206 confirm faulted/cancelled count as terminal for compounds
**Exploration:** quick → revised under review
**Depends on:** D6 (QuorumConfig relationship determines how threshold config is expressed)
**Status:** revised (R1-07: corrected false claim about consistency with compound M-of-N; R1-08, R1-20: added unreachable-quorum detection)

## D3: Straggler policy — KEEP and CANCEL from day one

**Choice:** Support both KEEP (let remaining bindings run) and CANCEL (cancel remaining bindings once threshold met) from day one, defaulting to KEEP. Reuse the existing `OnThresholdReached` enum from `io.casehub.api.model.OnThresholdReached`.
**Alternatives:**
- KEEP only, defer CANCEL — the original D3 choice. But CANCEL is not a future feature: `OnThresholdReached.CANCEL` has 11 references across the codebase including production use in `SubCaseCompletionService.java` line 173 (`cancelRemainingChildren(group, childCaseId)`), persistence tests, YAML mapping, and example cases. Deferring an already-implemented capability creates an inconsistency where SubCase M-of-N has CANCEL but binding-level quorum does not.
- CANCEL only — saves resources but loses potential data from slower agents.
**Rationale:** The infrastructure exists: `OnThresholdReached` enum with KEEP and CANCEL, `SubCaseCompletionService` demonstrating the CANCEL pattern with `cancelRemainingChildren()`. Supporting both from day one is the right design — the `requires` guard carries `onThresholdReached` from the existing enum, providing consistent behavior across SubCase and binding-level quorum.
**Trade-offs:** Implementing CANCEL at the binding level requires plan item cancellation logic in PlanningStrategyLoopControl. This is bounded work — the pattern exists in SubCaseCompletionService.
**Sources:** OnThresholdReached enum (api module), SubCaseCompletionService line 173 (CANCEL implementation), QuorumConfig record (carries onThresholdReached), 11 references to OnThresholdReached.CANCEL across codebase
**Exploration:** quick → revised under review
**Depends on:** D2 (threshold semantics determine what "remaining" means)
**Status:** revised (R1-10: CANCEL is production code, not deferred feature)

## D4: Scope — include compound MofN YAML wiring fix

**Choice:** Fix compound-level MofN YAML wiring within this issue
**Alternatives:**
- Separate issue — cleaner scope but leaves a silent bug while we're already working in the same area
**Rationale:** Small delta. `YamlCompound.mofnRequired()` is dropped during `YamlCaseDefinitionConverter.convertCompound()` — the converter passes `yc.completionSemantics()` but ignores `yc.mofnRequired()`. No production `BlackboardPlanConfigurer` bridges `CompoundDeclaration` to runtime `PlanItemDefinition.Compound`. Fixing this while adding the binding-level quorum keeps the "M-of-N coordination" story coherent. The two features share the M-of-N arithmetic concept but operate at different evaluation points: compound completion is evaluated by `DefaultCasePlanModel.evaluateCompletion()`, while binding-level quorum is a dispatch-pipeline gate in `PlanningStrategyLoopControl`.
**Trade-offs:** Slightly larger PR. Mitigated by the fact that both changes are in the same modules.
**Sources:** YamlCaseDefinitionConverter.convertCompound() (line 1315), YamlCompound.mofnRequired() field, BlackboardPlanConfigurer SPI (no production implementations), DefaultCasePlanModel.evaluateCompletion()
**Exploration:** quick
**Status:** captured (rationale corrected per R1-13: compound and binding quorum are different evaluation points, not "same runtime machinery")

## D5: Runtime hook — pre-strategy filter in PlanningStrategyLoopControl

**Choice:** Evaluate `requires` guards in PlanningStrategyLoopControl after compound gating but before strategy dispatch
**Alternatives:**
- Inside PlanningStrategy — each strategy evaluates barriers. More flexible but every strategy must handle it.
- New LoopControl decorator — isolation and testability, but the concerns have ordering dependencies: compound gating must precede quorum gating (you must know the compound is active before checking its bindings' quorum states), and quorum gating must precede strategy dispatch. These are pipeline steps with dependencies, not independently composable concerns. A decorator chain obscures the ordering contract that the linear pipeline makes explicit.
**Rationale:** PlanningStrategyLoopControl.select() already implements a linear dispatch pipeline: compound lifecycle evaluation → scope-activated binding collection → compound gating → implementation routing → strategy dispatch → dispatch indexing. Adding `requires` guard evaluation as a step after compound gating fits naturally: bindings whose prerequisites are not yet met are filtered from the eligible set. When a binding completes and potentially satisfies a `requires` guard, the runtime re-evaluates blocked bindings on the next `select()` invocation. Since D1 now uses only `requires` (no `quorumBarrier` trigger), there is a single filter path — no ambiguity about where guard vs. trigger evaluation happens.
**Trade-offs:** Strategies cannot factor quorum into priority decisions. Acceptable — quorum is a hard gate, not a soft preference.
**Sources:** PlanningStrategyLoopControl.select() dispatch pipeline, CompoundStrategyDispatcher, CompoundLifecycleEvaluator
**Exploration:** quick
**Depends on:** D1 (requires guard is the sole mechanism — single filter path)
**Status:** captured (updated per R1-15: added ordering argument for pipeline steps; updated per R1-16: single mechanism resolves filter path ambiguity)

## D6: QuorumConfig relationship — lean binding-level model, reuse OnThresholdReached

**Choice:** Build a lean binding-level quorum model separate from `QuorumConfig`. Reuse the `OnThresholdReached` enum from `io.casehub.api.model`.
**Alternatives:**
- Extend QuorumConfig to binding level — `QuorumConfig` has four fields: `instances` (how many sub-cases to spawn), `required` (threshold), `onThresholdReached` (straggler policy), `allowSameAssignee` (assignment constraint). Only `required` and `onThresholdReached` apply to binding-level quorum. `instances` doesn't map — binding-level quorum coordinates already-declared bindings, it doesn't spawn new ones. `allowSameAssignee` is a SubCase assignment concern with no binding-level equivalent. Extending QuorumConfig forces two inapplicable fields onto every binding quorum declaration.
- Replace QuorumConfig — binding-level model becomes the canonical quorum type, deprecate `QuorumConfig`. Too disruptive — `QuorumConfig` is wired through YAML conversion, persistence, and SubCase infrastructure. The two models serve genuinely different purposes (spawning vs. coordination).
- Keep both without explicit relationship — the original (implicit) approach. Creates confusion about which to use when.
**Rationale:** Binding-level quorum needs: `bindings` (which bindings to wait for), `required` (threshold count), `onThresholdReached` (KEEP or CANCEL). This is structurally different from `QuorumConfig` which models instance spawning. The `OnThresholdReached` enum is the shared concept — reusing it ensures consistent straggler semantics across SubCase and binding-level quorum. A new record (e.g., `BindingQuorum` or embedded in the `requires` field) carries only the binding-level fields.
**Trade-offs:** Two quorum-related types in the API. Mitigated by clear naming and distinct use cases: `QuorumConfig` for SubCase instance quorum, binding-level model for coordination barriers.
**Sources:** QuorumConfig record (api/spi/QuorumConfig.java): instances, required, onThresholdReached, allowSameAssignee. OnThresholdReached enum (api/model/OnThresholdReached.java): KEEP, CANCEL.
**Exploration:** surfaced during review
**Status:** captured (new decision per R1-04, R1-18)
