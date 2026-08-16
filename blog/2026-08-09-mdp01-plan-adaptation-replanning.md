---
layout: post
title: "Plan Adaptation — When Plans Meet Reality"
date: 2026-08-09
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [planning, adaptation, spi, concurrency, casehub]
---

The engine's goal decomposition creates a plan at case start — but plans go stale the moment a worker returns with unexpected results. Plan adaptation closes that gap: after each worker completion, the engine evaluates whether the remaining plan still makes sense and, if not, asks the LLM to revise it.

The architecture splits the problem into two orthogonal SPIs. `AdaptationTrigger` answers "should we replan?" — the built-in `EveryStepTrigger` always says yes, while `OnFailureTrigger` fires only when something went wrong. `PlanRevisionStrategy` answers "what should the new plan be?" — `ForwardReplanRevision` sends the completed step history and current context to the LLM and asks for the remaining steps. Both are `NamedStrategy` implementations, resolved per case definition via `EngineStrategyResolver`. Consumers can plug in their own without touching the orchestrator.

The interesting design constraint was compound replacement. Initial decomposition creates a complete compound via `registerDefinition()` — it populates `scopedBindings`, `childrenIndex`, and `definitions` in one atomic operation. Adaptation can't use the same path because the compound is live: some PlanItems are COMPLETED, some might be RUNNING, and pending ones need to be marked OBSOLETE before the new steps are materialised. `CasePlanModel.replaceCompound()` handles this — it unregisters old children, preserves completed and running items, registers the new compound definition, and bumps a generation counter for idempotency.

Concurrency needed careful thought. Adaptation runs synchronously because `CompoundCompletionEvaluator` must evaluate the revised plan, not the stale one. But synchronous LLM calls on worker threads risk pool starvation. A `Semaphore` with three permits bounds concurrent adaptations across all cases. Within a single compound, a `ReentrantLock` prevents two near-simultaneous step completions from both triggering revision — the generation counter catches the second one after it acquires the lock.

The YAML surface supports both explicit configuration and presets:

```yaml
spec:
  decompositionStrategy: llm
  adaptation: adaptive          # every-step + forward-replan
```

or the more conservative variant that only replans on failure:

```yaml
  adaptation: conservative      # on-failure + forward-replan
```

One latent issue surfaced during implementation: `PlanItemDefinition.Primitive`'s compact constructor validates `Objects.requireNonNull(executor)`, but structural Primitive children — the ones that exist only as compound index structure — don't use executor at all. `DefaultGoalDecomposer` passes null and gets away with it because the validation was added later. The adaptation code hit the NPE immediately. The workaround is a placeholder `ExecutorRef`, but the real fix is separating structural from executable Primitives.

With adaptation in place, decomposed plans are no longer static. A case that starts with three steps can end with five — or two — depending on what the workers discover along the way. This is the foundation that goal revision will build on: once the engine can revise plans, revising the goals that generated those plans is the natural next step.
