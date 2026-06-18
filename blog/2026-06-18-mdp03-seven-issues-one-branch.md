---
layout: post
title: "Seven issues, one branch — provisioning through to Qhorus"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Engine]
series: issue-530-provisioning-binding-enhancements
---

The failure cascade shipped in the previous session but had no connection to Qhorus. An agent declining via a DECLINE speech act on a channel wrote a generic `channelMessage` signal — the same as a DONE or RESPONSE. The engine had no way to distinguish "I'm done" from "I refuse."

Seven issues covered the gap, from the provisioner surface up to the Qhorus bridge.

## The small ones that cleared fast

`ProvisionContext` needed `tenancyId` so provisioners could resolve tenant-specific endpoints. One field on a record, one construction site, seven call sites to update — the kind of change where the breakage is the point, because every caller now explicitly passes the tenant identity.

The `getCapabilities()` gate in `tryProvision()` was blocking endpoint-registered capabilities from ever reaching the provisioner. The gate checked a static set; endpoints registered at runtime were invisible. We removed the gate and let `provision()` decide — it has the full context now, including `tenancyId`.

`HumanTaskTarget` gained an `outcomes` field — `Set<String>` propagated through to `WorkItemCreateRequest.permittedOutcomes`. Without this, a typo in a gate reviewer's outcome name creates a dead case with no error.

## Binding extensions for failure cascades

Two additions to `Binding` that the failure cascade needs but didn't have.

`inputSchemaOverride` lets a binding dispatch the same capability with narrower input. When a full security review fails and the cascade falls to a reduced-scope retry, the retry binding references the same capability but overrides the input JQ to select only the flagged files. Without this, creating a separate `security-review-reduced` capability would fork the trust scoring path — wrong, because it's the same skill at different scope.

`contextWrite` applies key-value writes to the case context before dispatch. The failure cascade problem: a Tier 3 binding's condition stays true after it fires, because the context hasn't changed yet. On the next `CONTEXT_CHANGED`, the condition re-evaluates, the binding fires again — infinite loop. `contextWrite` lets the binding mark `reducedScope: true` and reset `status: PENDING` before dispatching, breaking the loop.

## DEEP_MERGE

Worker output was replacing entire blackboard keys. When a capability key held failure tracking state — `attempts`, `history`, `excludedAgents` — a successful completion destroyed the audit trail. We extracted `ConflictResolver` as a shared utility in `api` with four strategies: `LAST_WRITER_WINS`, `FIRST_WRITER_WINS`, `FAIL`, and `DEEP_MERGE`. The deep merge recursively preserves existing map keys that the incoming output doesn't overwrite.

The humanTask output path had the same problem — `PlanItemCompletionApplier` used `setAll()` with no strategy awareness. It now looks up the binding's `conflictResolverStrategy` via the `CaseDefinitionRegistry` and applies per-key resolution through the same `ConflictResolver`.

## The Qhorus bridge

The interesting design question: when a Qhorus DECLINE arrives, how does it enter `handleSemanticFailure`? The bridge observes `MessageReceivedEvent` on a managed CDI thread. It has a `correlationId` — the `eventLogId` of the original COMMAND. From the EventLog metadata: `workerName`, `bindingName`, `inputDataHash`. From the case instance repository: the running case. That's everything `WorkflowExecutionCompleted` needs.

The edge cases required more thought than the happy path. A non-numeric `correlationId` means the DECLINE wasn't responding to an engine-dispatched COMMAND — fall through to the existing signal path. An EventLog that doesn't exist means the same thing. A missing case instance means the case already completed — skip silently. A worker not in the current definition means the definition changed since dispatch — construct a minimal `Worker` from the EventLog metadata, because the failure cascade only needs the name.

The bridge now forks cleanly: DONE and RESPONSE still write `channelMessage` signals; DECLINE and FAILURE publish `WorkflowExecutionCompleted` with the appropriate `WorkerOutcome` and let the existing cascade handle the rest.
