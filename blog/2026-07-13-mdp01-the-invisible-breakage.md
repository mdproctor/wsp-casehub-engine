---
layout: post
title: "The Invisible Breakage"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [snapshot-drift, intellij-mcp, sla, jsonschema2pojo]
---

I set out to fix four small issues on the engine — the kind of cleanup batch where you expect to be done in an hour. Two constructor mismatches from upstream SNAPSHOT changes, one checkstyle violation, and a TODO that had been waiting for someone to replace it with an actual event dispatch.

The constructor fixes were straightforward: `CommitmentContext` grew three parameters in qhorus-api, `TrustRoutingPolicy` gained a `Set<TrustPhase>` parameter. But neither breakage showed up until I ran a full project build. `mvn install -DskipTests` passed clean — production code compiled fine. The breakages lived exclusively in test files that hadn't been recompiled since the upstream SNAPSHOTs changed. Without a cross-repo CI gate on test compilation, SNAPSHOT drift is invisible until someone touches the module.

The more interesting trap was IntelliJ's build scope. `ide_build_project` compiles everything the IDE has open — not just the Maven reactor for the current project. I saw `TrustRoutingPolicy` errors in `RoutingSupportTest`, `CbrAgentRoutingStrategyTest`, and `LlmAgentRoutingStrategyTest`, fixed them via IntelliJ's text replacement API, then found none of the changes appeared in `git diff`. Those files were in casehub-blocks, a sibling project open in the same IDE window. The fixes went to the right files in the wrong repo.

The SLA violation fix was the one piece with actual design surface. `MilestoneActivatedEventHandler` had a TODO where the deadline had already passed: log a warning, return void. The fix publishes `MilestoneSLAViolatedEvent` immediately — the same event the Quartz timeout job would have fired. Past-deadline SLA is a common edge case in event-driven scheduling: if you only handle the future path, any activation delay that pushes past the deadline silently drops the violation.

The GoalExpression schema update was a one-line change with a telling side effect. Making `allOf`/`anyOf` items accept either strings or nested `GoalExpression` objects (recursive `$ref` in JSON Schema) caused jsonschema2pojo to generate `List<Object>` instead of `List<String>`. Existing tests passed without modification — Java's type inference handles `List.of("goal-a")` in a `List<Object>` parameter context — but the generated API is now less discoverable. Schema accuracy won over API ergonomics; the engine's internal `GoalExpression` sealed interface is the real API anyway.
