---
title: "Scope and the Silent Guard"
date: 2026-05-23
entry_type: note
subtype: log
projects: [casehub-engine]
tags: [casehub, humanTask, SLA, contextChange]
excerpt: >
  Two small fixes that close a larger gap — scope propagation for SLA preference
  routing and a silent when-field bug in contextChange bindings.
---

Two issues, both discovered via devtown and clinical integration work. Neither
was complex to fix. Both were invisible until the right test ran.

**engine#330** — `HumanTaskScheduleHandler` creates WorkItems without setting
`scope`. That means `ExpiryLifecycleService.buildBreachContext()` always
resolves preferences at `Path.root()` for HITL tasks. The application's
`SlaBreachPolicy` gets org-level defaults instead of case-type preferences.

The fix is plumbing: add `scope` as an optional string to `HumanTaskTarget`,
wire it through the YAML schema and mapper, set it on
`WorkItemCreateRequest.scope()` (inline mode) and `workItem.scope` (template
mode). No new abstractions, no new dependencies. The scope string follows the
existing platform convention — `"casehubio/devtown/pr-review"` — and the work
module's expiry service already knows what to do with it.

I kept `scope` as a raw `String` rather than the platform's `Path` type.
`HumanTaskTarget` lives in `casehub-engine-api`, a pure-Java tier. Adding a
`casehub-platform-api` dependency there would create cross-foundation coupling
for what amounts to a label. The contract is documented; enforcement belongs
downstream.

**engine#335** — `CaseContextChangedEventHandler` evaluates
`on.contextChange.filter` but silently ignores `binding.getWhen()`. The `when`
field only worked for schedule/timer triggers. A contextChange binding with
`when: ".requiresDsmbEscalation == true"` fired unconditionally on every
context change.

Four lines. Same evaluation pattern as `SchedulerService`. The dedup guard
("PlanItem is not PENDING, skipping") was masking the problem — the wrong
binding fired but the guard prevented a duplicate WorkItem. The initial
creation was still wrong.
