---
layout: post
title: "The Tree That Writes Itself"
date: 2026-08-31
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [yaml, htn, decomposition, planning]
---

# The Tree That Writes Itself

HTN decomposition in CaseHub was runtime-only. GOAP evaluates preconditions and effects to find a plan. LLM decomposition prompts a language model. Portfolio cascades through delegates. All three produce the task tree dynamically — there's no way to say "here's the tree, just run it."

That's fine when the decomposition genuinely needs reasoning. It's wrong when you already know the answer. An incident response playbook has a known branching structure: high severity goes through triage, escalation, and resolution; low severity skips to auto-resolution. The guard is a JQ expression on the case context. The task list is fixed. There's nothing to plan — the human already did the planning when they wrote the playbook.

The YAML surface for this is a `spec.decomposition:` block with a root compound task, guarded methods, and leaf tasks referencing capabilities:

```yaml
spec:
  decomposition:
    root:
      name: investigate-incident
      methods:
        - guardLabel: "High severity"
          guard: ".severity == \"high\""
          tasks:
            - name: triage
              capability: triage-assessment
            - name: escalate
              capability: escalation
        - guardLabel: "Low severity"
          tasks:
            - name: auto-resolve
              capability: auto-resolution
```

Nesting is arbitrary. A task node with `capability:` is a leaf. A task node with `methods:` is a compound that decomposes further. Guards are JQ expressions evaluated against the working layer — same surface as binding conditions.

The type mapping is direct: `YamlHtnNode` → `CompoundTask<JsonNode>` or `HtnLeafTask`, `YamlHtnMethod` → `DecompositionMethod<JsonNode>`. The converter walks the tree recursively, producing `CompoundTask` instances with inline strategies that flatten to `DagPlan<LeafTask>` at runtime. `ExplicitHtnDecompositionStrategy` (id=`"explicit"`) evaluates method guards and delegates — no search, no LLM, no planning. The tree IS the plan.

What made this clean was the existing type system. `TaskNode` is a sealed interface with `LeafTask` and `CompoundTask`. `DecompositionMethod` already carries a `Predicate<T>` guard and a `DecompositionStrategy<T>`. The YAML records are a thin layer over types that already existed for the runtime strategies — the same hierarchy that GOAP and LLM produce dynamically, we now produce declaratively.
