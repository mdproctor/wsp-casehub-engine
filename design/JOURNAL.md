# Design Journal — issue-321-324-blackboard-fixes

### 2026-05-22 · §Execution Models · §Worker Execution Lifecycle

`PlanItemStatus.RUNNING` previously conflated two semantically distinct active
states: a Quartz job actively computing (CapabilityTarget) and the engine waiting
for an external actor to respond (SubCase, HumanTask, Extension). The ambiguity
was silent but meaningful — an LLM or observer reading RUNNING on a SubCase
PlanItem would infer local computation when the engine is actually idle, waiting
for a child case to return a signal.

The new DELEGATED state resolves this: RUNNING means a Quartz job is executing;
DELEGATED means control has passed to an external actor and the engine waits for
a completion signal. The transition rules are PENDING→RUNNING (CapabilityTarget
only), PENDING→DELEGATED (SubCase, HumanTask, Extension), and PENDING→FAULTED
for pre-dispatch errors. Terminal transitions accept both RUNNING and DELEGATED
as source states.

This required closing a correctness gap: SubCase and HumanTask PlanItems could
previously be left stuck in PENDING or DELEGATED with no recovery path when
handlers encountered errors (spawn failure, missing CaseDefinition, M-of-N group
rejection). All error paths now fault or cancel the PlanItem explicitly.

SubCase completion now routes through PlanItemCompletionHandler via the new
SubCaseExecutionCompleted event, ensuring stage autocomplete fires for SubCase
bindings using the same path as worker completion. The BlackboardRegistry
completion index was renamed from the workerName-keyed API to a generic
trackingKey API (childCaseId for SubCase, workerName for CapabilityTarget),
ready for HumanTask when its completion path is wired.
