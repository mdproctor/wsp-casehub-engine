---
layout: post
title: "Giving the YAML a Human Concept"
date: 2026-05-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [hitl, yaml-dsl, quarkus]
---

The `casehub-engine-work-adapter` has been implemented for weeks. `HumanTaskScheduleHandler` creates WorkItems. `WorkItemLifecycleAdapter` listens for completion and signals the engine to resume. The code is there. But devtown's PR review case stalls every time the human approval step fires.

The first assumption — that the adapter simply wasn't on the classpath — was wrong. The problem was upstream of that.

Devtown's `pr-review.yaml` had `human-approval` defined as a `capability` binding:

```yaml
- name: human-approval
  capability: "human-decision:pr-approval"
```

The `capability` path routes to WorkerProvisioner — Quartz, worker scheduling, function execution. `HumanTaskScheduleEvent` is never published on this path. The adapter never gets a chance to run regardless of what's on the classpath.

So the question was: how should human-in-the-loop steps be expressed in the YAML DSL? Two options. Leave `capability` as the only binding target type and rely on a naming convention (`human-decision:*`) — every harness implements its own bridge. Or add `humanTask` as a first-class binding type with the schema explicitly modelling what a human task needs: title or template reference, input and output mappings, candidate groups, expiry.

I preferred the second. Convention-based approaches break silently — a typo in the capability name produces no error, the WorkItem is never created, and the case stalls with nothing to diagnose. A first-class type means the schema rejects invalid configurations before the engine ever starts. The mapper routes it directly to `HumanTaskScheduleHandler`. Consumers get IDE completion. That's the direction I want the platform going.

We extended the JSON Schema with a `HumanTask` definition — mutually exclusive `title` (inline mode) and `templateRef` (template mode), optional mappings and candidate constraints. `CaseDefinitionYamlMapper` gained a `convertHumanTask` method. The YAML for the PR review case became:

```yaml
- name: human-approval
  humanTask:
    title: "PR approval required"
    outputMapping: "{ humanApproval: { status: .decision } }"
```

One thing Claude caught during review: `jsonschema2pojo` generates absent list fields as empty `ArrayList`, not null. The `!= null` guard alone wasn't enough — explicit `candidateGroups: []` and no `candidateGroups` at all both produce the same empty list after Jackson deserialises. Both should map to null on the built target. Needed `!= null && !isEmpty()`.

Wiring the adapter into devtown's runtime turned up two pre-existing failures. All `@QuarkusTest` classes fail because qhorus's `ExcludedTypeBuildItem` gates reactive REST resources from CDI but not from the JAX-RS scanner, which discovers `@Path` beans independently — duplicate `GET /.well-known`. The fix is in qhorus (their ADR-0007); filed as devtown#35. Production augmentation also fails because `casehub-persistence-hibernate` is absent from devtown's assembly — the in-memory stores work for tests but build-time augmentation needs JPA implementations for the SPI repositories. Filed as devtown#34.

Neither is ours to fix. What is done: `human-approval` now speaks `humanTask`, the adapter is wired, and devtown#30 — the end-to-end HITL integration test — is unblocked.
