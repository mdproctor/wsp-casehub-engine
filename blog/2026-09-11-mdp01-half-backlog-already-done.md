---
layout: post
title: "Half the Backlog Was Already Done"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [backlog, schema, reasoning, cbr]
---

# Half the Backlog Was Already Done

I pulled every S and XS issue off the engine backlog — eight issues, small enough to batch onto one branch. The plan was a quick sweep through the bottom of the queue.

Four of the eight were already implemented. Schema validation tests that had been fixed in a prior commit. ShorthandModule adoption that landed weeks ago. The ActorState capacity override already wired. Reasoning propagation already reconciled. Each one sitting open on GitHub while the code was already on main.

This happens when implementation outruns issue hygiene. A feature lands as part of a larger commit, the focused issue stays open, and the backlog accumulates phantom work. The fix is closing issues at commit time, not in a separate pass — but when you're deep in a multi-issue epic, the small ones slip.

The three that needed real work were small but real. Schema validation now covers every YAML test fixture across the project, not just the examples directory. The test filters by `dsl:` field presence to distinguish full case definitions from fragments — a pattern that avoids hardcoded fixture lists. Ten fixtures needed fixing: semver versions, kebab-case names, a stale `expression` field that should have been `filter`.

The more interesting change was splitting importance weights for the worker-reasoning memory domain. `ReflectionTriggerConfig` calibrates weights for when to trigger reflection — SUCCESS at 0.3 because a success doesn't need much reflection. But for reasoning retrieval, a successful security review's reasoning ("APPROVED because X, Y, Z") is highly actionable. `MemoryRetrievalConfig` now carries its own weights: SUCCESS at 0.7 instead of 0.3. The two systems have different goals — reflection is about when to pause and think; retrieval is about what's worth remembering.
