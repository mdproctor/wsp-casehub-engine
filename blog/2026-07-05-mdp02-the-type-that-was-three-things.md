---
layout: post
title: "The Type That Was Three Things"
date: 2026-07-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [platform-convention, classification, path, types, labels]
series: issue-652-cross-repo-blocker-batch
---

*Part of a series on [#652 — add semantic labels/tags to CaseDefinition Java API](https://github.com/casehubio/engine/issues/652). Previous: [Two Canonicals, One Convention](2026-07-05-mdp01-two-canonicals-one-convention.md).*

The issue asked for three things: labels, tags, and categories on `CaseDefinition`. The YAML schema already had `tags` — a `type: object` key-value map that had never been wired to the Java API, never consumed by any code, never used in a single consumer YAML definition. Dead on arrival from the original Serverless Workflow scaffold. `metadata` was the same: defined in the schema, mapped by jsonschema2pojo, called by nobody.

The real question wasn't what to add. It was what the platform actually needed for case classification.

casehub-desiredstate wants to say "this case is a situation-response of type replan" — a type declaration, not a tag. casehub-work already has `Path`-based labels on WorkItems through `LabelDefinition` and `WorkItemLabel`. The platform has the `Path` record in `casehub-platform-api` with `isAncestorOf`, `parent`, `depth`. The primitive exists. The convention didn't.

Three concepts collapsed into two once I stopped thinking about them as separate features and started thinking about what each one answers. `types` answers "what contracts does this case implement" — behavioural classification that affects routing, dispatch, completion. `labels` answers "how is this case organised" — operational classification for queues, dashboards, analytics. Both are `Set<Path>`. Types use implements-semantics: a case definition can be both `situation-response/replan` and `compliance/auditable` across orthogonal dimensions. Each path provides the vertical hierarchy; the set provides horizontal breadth.

The `Set<Path>` decision for types came from a specific question: is this `extends` or `implements`? A single `Path` would force every case into one type hierarchy — `situation-response/replan` but not also `compliance/auditable`. Those are independent type dimensions. Multi-valued types with `Path` hierarchy within each dimension gives the expressiveness without inventing a new primitive.

The design review caught thirteen issues across three rounds. The ones that mattered: the spec claimed `CaseDefinition` fields were "immutable, builder-constructed" — they're not, the class is mutable with public setters and the mapper constructs via `new` plus setters. The spec referenced `registeredDefinitions()` — a method that doesn't exist; the registry's internal structure is `Map<CaseKey, RegistryEntry>`. The spec claimed types and labels would be "persisted implicitly in the `definition` JsonNode column" — that column is never populated by `registerCaseDefinition()`. Each of these would have become a bug discovered during implementation. Finding them in the spec cost nothing.

The platform convention — `types: Set<Path>` and `labels: Set<Path>` on every definable entity — shipped with `CaseDefinition` as the first adopter. `WorkItem`/`WorkItemTemplate` is the second, tracked in work#291. The convention went into PLATFORM.md before the second adopter exists, because stating the pattern early is how platform coherence works. If the second adopter reveals a problem, the convention adapts — but having it written down means the adaptation is deliberate, not accidental.

The dead `tags` and `metadata` properties are gone from the schema. The `required` list lost `metadata`. The document-processing example lost its vestigial Kubernetes-style `apiVersion`/`kind`/`metadata` fields and gained `types` and `labels`. Cleaning up while you're there costs less than a dedicated cleanup issue later.
