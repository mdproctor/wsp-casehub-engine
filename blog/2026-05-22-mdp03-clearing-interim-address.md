---
layout: post
title: "Clearing the Interim Address"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [refactoring, platform, quarkus, cdi]
---

When we moved `JQEvaluator` into `casehub-engine-common` as part of the JQ
consolidation a few days ago, we knew it was temporary. The comment in CLAUDE.md
said so explicitly: *follow-on platform extraction tracked in engine#317.* The
right home for a domain-agnostic JQ evaluator is Foundation tier — somewhere
`casehub-work`, `casehub-qhorus`, and anyone else can reach it without taking a
dependency on Orchestration. Today's branch finished that move.

Before starting, I checked PR #313 — the json-schema-validator pin fix that was
supposedly still open. It wasn't. The commit was already on main (`953f725`), the PR
just hadn't been closed. GitHub showed head and base both as `main`, which is the
signature of a PR created from the wrong branch or after the work landed directly.
I closed it and moved on.

The actual migration was forty files — six source files deleted from `engine-common`,
thirty-four import sites updated across `runtime`, `scheduler-quartz`, `blackboard`,
and `work-adapter`. The import mapping was straightforward:

```
io.casehub.engine.internal.jq.JQEvaluator      → io.casehub.platform.expression.JQEvaluator
io.casehub.engine.internal.config.SecretManager → io.casehub.platform.api.expression.SecretManager
```

The pom change was two entries in the root `<dependencyManagement>` block and two
lines in `common/pom.xml`. The IntelliJ MCP handled the text replacements cleanly.

What it didn't catch was `ConfigContextIntegrationTest`. That test class lives in
`io.casehub.engine.internal.config` — the same package as the deleted `ConfigManager`
and `SecretManager`. No import statement. Claude had been searching for
`import io.casehub.engine.internal.config.*` and found nothing, because same-package
references don't need imports. The build caught it on the first compile attempt;
the fix was two added imports. Worth remembering: after any package-level deletion,
also check for test classes living in the same package as the deleted types.

The other surprise was `ValidationResultTest`. The platform's `ValidationResult`
record has a compact constructor that rejects `null` error messages with an
`IllegalArgumentException`. The engine's old version accepted them silently. One test
was named `error_nullMessageDoesNotThrow` — documenting behaviour that the platform
deliberately doesn't preserve. I updated it to `error_nullMessageThrows` and verified
the new contract makes more sense anyway: an error result with no message is useless.

Four test configs needed `quarkus.index-dependency` entries for
`casehub-platform-expression`. This is the standard library JAR CDI discovery
pattern — Quarkus doesn't index transitive dependencies automatically, so any
`@ApplicationScoped` bean in a platform artifact needs an explicit entry wherever
`@QuarkusTest` is used. It's mechanical but easy to miss when adding a new platform
dependency.

The runtime `@QuarkusTest` suite is still blocked by a pre-existing
`RoutingCursorStore` unsatisfied dependency — a separate fix. The migration itself
compiles clean and the non-Docker tests pass.
