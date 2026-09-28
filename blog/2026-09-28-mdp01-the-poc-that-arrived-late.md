---
layout: post
title: "The PoC That Arrived Late"
date: 2026-09-28
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [architecture, caseflows, framework-neutrality, durable-execution]
---

# The PoC That Arrived Late

The Caseflows Blueprint landed on my desk as a proof-of-concept exploring architectural ideas for CaseHub. Its central premise: CaseHub is too tightly coupled to Quarkus. Framework integrations sit at the composition boundary in Caseflows — the engine core is pure Java, and Spring Boot and Quarkus are thin adapter shells.

The problem is, CaseHub already did this. The triple-module refactoring — `-core` for neutral Java, bare-name for Quarkus, `-spring` for Spring Boot — spans 60+ modules in engine and 150+ in platform. The `api` module compiles with zero Quarkus, zero JPA, zero reactive types. `ArchitecturalExclusionTest` enforces the dependency rule. Platform ships Spring Boot starters and code generators.

So the PoC's most prominent contribution — framework neutrality — was independently delivered in the main codebase while it was being written. That doesn't make it worthless. It makes the question sharper: which of its *other* ideas carry enough weight to adopt?

I found four worth evaluating, in descending order of defensibility.

**Durable execution primitives** — fencing tokens, execution leases, a durable inbox with settlement. CaseHub's resilience module covers DLQ, retry, and replay, but not the formal lease/fence model for multi-node crash recovery. The lease types are clean and self-contained. The tension: adopting them means CaseHub acknowledges that multiple nodes might evaluate the same case — which shifts the deployment model from single-owner to lease-arbitrated. That's a fundamental assumption change, not just an SPI addition.

**CaseState separation** — splitting engine-owned lifecycle state (status, goals, milestones) out of `CaseContext` into a read-only `CaseState` record. CaseHub already has the `$case` JQ convention. Making it a type-level boundary prevents workers from accidentally writing engine state. The cost is a broad migration: every test that reads lifecycle state through context assertions needs updating.

**Snapshot/patch protocol** — formalizing the worker context isolation that CaseHub already does informally. `CaseContextSnapshot` (immutable) → worker copy → `CaseContextPatch` (RFC 6902 diff). Useful for audit. The conflict policies it enables (LWW, FWW, deep merge) solve a problem CaseHub avoids by design — sequential evaluation via `CaseEvaluationSerializer`.

**The Kernel** — a case-ignorant execution substrate beneath the runtime. Architecturally the most ambitious idea, and the least justified. CaseHub's `-core` modules already achieve framework neutrality. The kernel's value depends on having a second client (desiredstate? qhorus?) that genuinely needs the same dispatch model. Neither maps cleanly onto addressed-command dispatch. The most likely outcome is a kernel with exactly one client, adding indirection for an architectural purity that serves no operational purpose.

The deepest finding from the code analysis: 98% of the Caseflows API surface is kernel-independent. Only 3 of ~170 source files import kernel types. The ideas are portable without the kernel — which argues against adopting it even more strongly.
