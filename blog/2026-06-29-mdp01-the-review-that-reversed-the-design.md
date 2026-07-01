---
layout: post
title: "The Review That Reversed the Design"
date: 2026-06-29
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [cmmn, goals, milestones, design-review]
---

## The Review That Reversed the Design

I started this branch thinking `Goal.terminal` just needed wiring. The field existed in the model, the schema, and the builder — it had never been connected by the YAML mapper and the handler ignored it. The brainstorming conclusion seemed obvious: wire it properly, gate completion evaluation on it, and move on.

The adversarial design review disagreed.

The reviewer went back to epic #84's actual text — not my interpretation of it — and pointed out the contradiction. The epic is unambiguous: "Non-terminal Goals are removed — they were Milestones." The spec I'd written was reversing the epic's core conclusion while claiming to close it.

The reviewer was right. A non-terminal goal IS a milestone with polarity. That's the overlap the epic was created to eliminate. Keeping `terminal` and wiring it would have preserved the exact conceptual confusion the epic identified, just with better plumbing.

The fix was cleaner than the original plan. Remove the field entirely. Goals are always terminal — they exist to drive case completion. If you need a non-terminal checkpoint with success/failure semantics, that's a milestone. `GoalExpression` membership is the sole mechanism for determining which goals gate completion. No second authority, no contradiction.

## What the Review Also Surfaced

The adversarial review found three gaps the brainstorming missed entirely:

**Milestone lifecycle was still interim.** `CasePlanModel` tracked milestones with a `ConcurrentHashMap<String, Boolean>` — achieved or not. The `MilestoneLifecycleStatus` enum (PENDING, ACTIVE, COMPLETED) already existed but nothing used it. The Javadoc explicitly said "interim approach — full alignment tracked in engine#84." Upgrading to the enum with proper transition guards and `ConcurrentHashMap.compute()` atomicity was straightforward, but I hadn't thought to include it in the spec.

**Two parallel milestone evaluation paths.** `CaseContextChangedEventHandler.milestones()` fired `MILESTONE_REACHED` on every context change where `completionCriteria` was true — no idempotency, no entry criteria check. Meanwhile `MilestoneLifecycleManager` ran the full lifecycle (PENDING→ACTIVE→COMPLETED), fired once per transition, and enforced entry criteria. Both ran on every context change. Removing the old path and making `MilestoneLifecycleManager` the sole evaluator eliminated duplicate EventLog entries and aligned with CMMN's requirement that a milestone pass through Available before Completed.

**`Milestone.parentStageId` didn't belong on the API model.** My spec put `parentStageId` on `Milestone` in the `api` module. The reviewer caught that Stage is a blackboard-internal construct — placing a stage reference on the public API model leaks a blackboard concept to every consumer. The back-pointer moved to `CasePlanModel.trackMilestone(name, stageId)` — the API model stays clean.

## The Conceptual Model

The CMMN alignment table that came out of this is the thing worth keeping:

- **Milestones** — you *pass* them. Neutral waypoints, no polarity, containable in stages. PENDING → ACTIVE → COMPLETED.
- **Goals** — you *achieve* them. Terminal outcomes with SUCCESS/FAILURE polarity. Drive case completion via `GoalBasedCompletion`.
- **Stages** — you *enter and exit* them. Containers for PlanItems and milestones.

No overlap. Each concept does one thing. The `terminal` field was the last remnant of confusion between the three, and removing it locks in the boundary.