# Design Journal — issue-237-lifecycle-scopes

## 2026-07-29 — Design + initial implementation

### Design decisions

**Scope declared on Binding, not Worker or PlanItemDefinition.** Binding is the dispatch control point — it already governs trigger conditions, input projection, outcome policy, and agent routing. Worker is a foundation-tier record in casehubio/worker; scope is deployment-time, not definition-time. PlanItemDefinition was rejected because it creates a parallel dispatch path.

**Sidecar model for completion.** Research across CMMN, Kubernetes (native sidecars GA v1.33), actor models (Akka), and BPMN multi-instance confirmed: separate scope lifetime from completion semantics. COMPANION workers excluded from evaluateCompletion, terminated after scope ends. PARTICIPANT workers count toward CompletionSemantics.

**CASE scope restricted to COMPANION only.** Case completion is goal-based (GoalBasedCompletion), not plan-item-based — no mechanism for a PARTICIPANT to block it. Workers that need to gate case completion should be COMPOUND-scoped PARTICIPANTs of a compound that gates the completion goal.

**Compound.scopedBindings → Map<String, Participation>.** evaluateCompletion needs participation metadata without external lookups to distinguish COMPANION from PARTICIPANT bindings.

**WorkerOutcome.Completed replaces context-key signaling.** The `_lifecycle.<bindingName>.done` approach was rejected — violates context layer model, untyped, invisible to compiler. Completed is a new sealed permit in WorkerOutcome, type-safe and discoverable.

**ExecutionMode.TRANSIENT (not SINGLE).** Design review renamed to describe actual semantics — no persistent session, no state continuity.

**QuartzWorkerExecutionJob suppresses WorkflowExecutionCompleted for non-TRANSIENT workers.** Scoped PlanItems stay RUNNING until worker signals Completed or scope terminates.

### What shipped

- casehubio/worker: WorkerOutcome.Completed, WorkerFunction.Persistent, PersistentScope, ScopeTerminatedException, WorkerScope.accumulatedState() — 53 tests green
- engine-api: LifecycleScope, Participation, ExecutionMode enums, ScopeActivatedTrigger, Binding extensions with build-time validation — 9 validation tests
- OutcomeKind.COMPLETED + all exhaustive switch fixes across WorkflowExecutionCompletedHandler, OutcomeKindTest, WorkerResultExpiredTest
- NoOpVocabularyRegistry.registeredUris() fix (eidos API update, unrelated)

### What's next

Tasks 3-6 and 8-9 from the implementation plan remain: Compound.scopedBindings Map change, ScopedWorkerRegistry + dispatch interception, Quartz completion suppression, lifecycle event handlers, PlanItem persistence + YAML, integration tests.
