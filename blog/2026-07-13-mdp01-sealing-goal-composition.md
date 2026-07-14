---
layout: post
title: "Sealing the cracks in goal composition"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [goal-expression, lifecycle-event, sealed-interface]
---

## Sealing the cracks in goal composition

I'd been looking at `GoalExpression` for a while, knowing the flat `allOf`/`anyOf` structure couldn't express what devtown actually needed — `anyOf(allOf(pr-approved, security-verified, ci-passing), externally-merged)`. Two distinct completion paths. The workaround was prepending `.pr.status == "merged" or` to every individual goal condition, which technically worked but polluted each goal's semantics. A goal called `pr-approved` shouldn't evaluate true when no approvals happened.

The fix was to make `GoalExpression` a sealed recursive tree. `SingleGoalExpression` holds a goal name. `AllOfGoalExpression` and `AnyOfGoalExpression` hold `List<GoalExpression>` children — themselves expressions, so nesting is natural. The sealed interface means evaluation is exhaustive: `isSatisfiedBy`, `goalNames`, and `satisfiedGoalName` are the only three operations, and every implementation must provide them.

The interesting part was `satisfiedGoalName`. The handler previously did a two-step dance — check if satisfied, then find the first matching goal name — with 30 lines of instanceof checks that duplicated the logic from the expressions themselves. Combining the check and the name extraction into one method collapsed that to a single call. For `AllOfGoalExpression`, it returns the first child's name only when *all* children are satisfied (null otherwise). For `AnyOfGoalExpression`, it returns the first satisfied child's name. The inversion is subtle but important — the design review caught that "same structure as AllOf" was misleading and made me show both implementations explicitly.

The other piece — enriching `CaseLifecycleEvent` — was more mechanical but addressed a real pain point. Every consumer that needed to know the case type or read the context was doing a reactive repository round-trip from an `@ObservesAsync` observer. Under `@Transactional`, that holds a JDBC connection for the duration. The fix: add `caseDefinitionName`, `namespace`, and `contextSnapshot` directly to the event record. A static factory on the record extracts everything from `CaseInstance`, handles the null-safety for meta model and context, and keeps the 13 fire sites clean. The design review surfaced that `QuartzWorkerExecutionJobListener` has no `CaseInstance` in scope — it runs in a Quartz thread with only `JobDataMap` strings — so an overloaded factory with null enrichment fields covers that case.

The YAML parser became recursive to support the nested composition. An array element can now be a string (goal name → leaf) or an object with `allOf`/`anyOf` (→ nested expression). Parse-time validation rejects unknown goal names and empty arrays — fixing a pre-existing null-safety gap where `goalMap.get(n.asText())` silently inserted null for undefined goals, producing an NPE at evaluation time instead of at parse time.
