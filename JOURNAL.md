# Design Journal — issue-1257-quorum-coordination-primitive

## 2026-10-27 — Session 1: Design + Batch 1-2.5 implementation

### Key insight: completion vs. dispatch

The issue assumed quorum didn't exist in the engine. It does — at four levels
(compound MofN, SubCase groups, action gates, HumanTask). The real gap is
binding-level dispatch barriers: "hold binding X until N of M other bindings
have COMPLETED." Compound MofN gates compound completion, not dispatch timing.

### Design decisions (6 captured, reviewed)

- D1: `requires` guard only — dropped `quorumBarrier` trigger. All four existing
  triggers respond to external events; quorum responds to internal plan model state.
- D2: COMPLETED-only threshold + unreachable quorum detection (SubCaseGroupPolicy formula).
- D3: KEEP and CANCEL from day one — reuse OnThresholdReached enum.
- D4: Include compound MofN YAML wiring fix — mofnRequired silently dropped.
- D5: Pre-strategy filter in PlanningStrategyLoopControl.
- D6: Lean binding-level model, reuse OnThresholdReached.

### Silent bug found: mofnRequired dropped

`YamlCaseDefinitionConverter.convertCompound()` reads `yc.mofnRequired()` via
the generated YAML record but never passes it to `CompoundDeclaration`. The field
was parsed and silently ignored. Additionally, no `BlackboardPlanConfigurer`
bridged YAML compounds to runtime — the YAML path was a dead end.

### Implementation progress

6 of 10 tasks complete:
- Batch 1 (API Foundation): BindingRequires record, validation, YAML schema/mapping — done
- Batch 2 (Compound Fix): mofnRequired wiring + CompoundBlackboardPlanConfigurer — done
- Batch 3 (Runtime): CasePlanModel metadata + RequiresBlackboardPlanConfigurer + filterByRequires in PlanningStrategyLoopControl — done
- Remaining: RequiresThresholdEvaluator (CANCEL/unreachable), @Requires annotation, CoordinationPipelineProjection, integration test
