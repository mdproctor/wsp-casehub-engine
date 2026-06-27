---
layout: post
title: "The Child That Didn't Write Home"
date: 2026-06-27
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [subcase, m-of-n, yaml, outputMapping]
---

## The Data That Disappeared

The M-of-N sub-case coordination in casehub-engine has been fully wired for weeks — group policies, threshold evaluation, cancellation on KEEP vs CANCEL. All working, all tested. But none of it was reachable from YAML. The four fields (`groupId`, `totalInGroup`, `requiredCount`, `onThresholdReached`) existed in the Java DSL's `SubCase.builder()` and were exercised by the runtime, but the YAML schema and mapper didn't know about them.

That gap blocked devtown's merge queue bisection. The queue needs two sub-cases spawned in parallel — left half and right half — each writing its result to a different key in the parent context. Without YAML support for grouped sub-cases, the case plan couldn't express it.

Wiring the schema was straightforward. Four optional properties in `CaseDefinition.yaml`, four lines in `CaseDefinitionYamlMapper.convertSubCase()`, sensible defaults matching the existing Java API behaviour. No surprises.

## The Silent Loss

The interesting part was the second bug. `SubCaseCompletionService.handleGroupedCompletion()` applies `outputMapping` — the JQ expression that maps a child's final context back into the parent. But it only applied the mapping for the child that triggered the M-of-N threshold. Every earlier child's output was silently discarded.

In a 2-of-3 group where all three children write to the same output key, this is invisible. The threshold-triggering child's value overwrites whatever the earlier children would have written. But in bisection — where child A writes `{ bisectLeft: .result }` and child B writes `{ bisectRight: .result }` — whichever child completes first loses its data permanently.

The code told the story clearly. `applyOutputMapping()` was called inside the `groupStatus == COMPLETED` block, after the threshold check. It only ran once, for the child whose completion tipped the count over `requiredCount`. The fix was moving that call before the threshold evaluation, gated on `childCompleted`. Every completing child now applies its mapping immediately. The threshold still controls parent resumption — it just no longer controls data flow.

## Data Flow vs Control Flow

The root cause was coupling two concerns in the same code path. The threshold decides when the parent can resume. The output mapping decides when data flows from child to parent. These are independent — a child's output is valid the moment that child completes, regardless of whether the group has reached its threshold.

Decoupling them was a three-line move. The `IN_PROGRESS` completion log now also records `appliedData` instead of `null`, which means the event stream captures per-child output even before the group completes. That's a free improvement for recovery — if the JVM crashes between child 1 completing and the threshold being reached, the applied data is in the event log.
