---
title: "The Field That Was Never Read"
date: 2026-08-12
author: mdp
projects: [casehub-engine, casehub-platform]
entry_type: note
subtype: diary
tags: [worker-rights, migration, silent-failures, yaml-parsing]
published: false
---

The platform generalized its worker rights SPI — `WorkerAction` from a closed enum to a record, `WorkerCredential` scoped by `ResourceId` instead of a bare UUID, authorization context as a marker interface instead of a hardcoded `caseDefinitionId`. Standard type-level work to make engine-specific vocabulary extensible across domains.

The engine migration was supposed to be mechanical. Replace enum constant references with a new `EngineWorkerActions` constants class, wrap case UUIDs in `ResourceId`, swap the engine's local `WorkerCredentialFilter` for the platform's reusable `acl-worker` module. Four hours of find-and-replace with a few new files.

What made it interesting was what we found along the way.

## The permissionIntent Gap

The YAML schema defines a `permissionIntent` field on bindings — a list of kebab-case action names like `read-context` and `signal-case` that declare what an automated worker is allowed to do. The schema validates it. The example YAML demonstrates it. The generated Java model stores it as `List<String>`.

The `CaseDefinitionYamlMapper` never converts it.

Every field in the generated `Binding` model gets mapped to the API `Binding` model in `convertBinding()` — capability, trigger, when expression, outcome policy, lifecycle scope, execution mode, exchange projection. But `permissionIntent` is silently skipped. The API model's field stays null, and the `CaseContextChangedEventHandler` hits this default:

```java
binding.getPermissionIntent() != null
    ? binding.getPermissionIntent()
    : List.of(EngineWorkerActions.READ_CONTEXT);
```

Every worker dispatch defaults to read-only access regardless of what the YAML author declared. A binding with `permissionIntent: [read-context, signal-case, write-context]` gets the same grants as one with no permission intent at all. The YAML author's security intent is silently discarded.

The fix was five lines — a `stream().map(EngineWorkerActions::fromKebabCase).toList()` call in the mapper, using the same kebab-to-constant lookup that `EngineWorkerActions` already provides. The interesting part isn't the fix. It's that the field existed in the schema, the examples, the generated model, and the API model — four layers deep — without anyone noticing the mapper never connected layer three to layer four.

## Fail-Closed by Default

The platform's `acl-worker` module ships with a `FailClosedWorkerScopeExtractor` as its `@DefaultBean`. It doesn't return `Optional.empty()` (which would skip scope validation). It returns a sentinel `ResourceId` that never matches anything. If you add `acl-worker` to your classpath without providing a real `WorkerScopeExtractor`, every worker request gets a 403.

This inverts the usual opt-in pattern. Most platform SPIs default to no-op — add the module, get nothing until you configure it. Here, adding the module actively blocks all worker requests until you provide the scope extractor. The insecure choice (no scope validation) requires a deliberate `PassthroughWorkerScopeExtractor` — you have to write the insecure thing on purpose.

The engine's `CaseScopeExtractor` is twelve lines — regex-match `cases/{uuid}` from the URL path, return `ResourceId`. But the pattern that forced it to exist is worth more than the implementation.
