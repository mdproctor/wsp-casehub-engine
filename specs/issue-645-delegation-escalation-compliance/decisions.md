## D1: Delegation eligibility policy

**Choice:** Structural signals — delegation is expected when the CaseDefinition provides structural decomposition paths for the binding (compound children, sub-case targets, decomposition strategy).
**Alternatives:**
- Outcome-only — observe whether agent delegated without assessing if warranted. Simple but noisy — agents correctly handling simple tasks get VIOLATED.
- Capability complexity — use multi-capability binding or sub-capability count. Vague, requires additional metadata.
**Rationale:** Follows the existing pattern where latency uses `AgentCapability.latencyHintP50Ms()` as a structural gate. Deterministic, uses existing CaseDefinition data, avoids false positives on simple leaf tasks.
**Trade-offs:** Case definitions without decomposition infrastructure get no delegation observation, even if delegation would have been appropriate in principle.
**Sources:** BehavioralComplianceRecorder.java:80-110 (latency gate pattern), ComplianceDimension.java (DELEGATION constant), BehavioralExpectations.java (delegationExpected API)
**Exploration:** quick
**Status:** captured

## D2: Escalation eligibility policy

**Choice:** PlannedAction presence — when the worker returned a PlannedAction on its outcome, the task was consequential. The PlannedAction declaration IS the escalation behavior.
**Alternatives:**
- All completions observed — every task completion by an escalation-expected agent is observed. Catches missed escalations but extremely noisy.
- Risk classifier registered — only observe when @RiskClassifier beans exist. Adds a system-level gate but doesn't change the per-task signal.
**Rationale:** Symmetric with delegation's structural approach. PlannedAction is the agent's explicit declaration that an action is consequential and needs oversight. Declining (WorkerOutcome.Declined) is also a form of escalation — the agent says "I can't handle this."
**Trade-offs:** Can only record COMPLIANT in v1 (agent escalated). Cannot detect when an agent SHOULD have flagged but didn't — unobservable at the engine level without additional instrumentation. Acceptable for v1; future versions can add WorkerRuntime-level hooks.
**Sources:** WorkerOutcome.java (Success.plannedAction()), ActionRiskClassifier SPI documentation, BehavioralExpectations.java (escalationExpected API)
**Exploration:** quick
**Status:** captured

## D3: Observation point

**Choice:** Single observation point at worker completion — keep all recording in BehavioralComplianceRecorder, called from WorkflowExecutionCompletedHandler on both success and failure paths.
**Alternatives:**
- Multi-point — add hooks in WorkerRuntime (delegation detection), ActionGate pipeline (escalation detection). Richer signals but touches multiple modules and breaks the single-recorder pattern.
**Rationale:** Consistent with LATENCY and ATTESTATION_RATE dimensions which both observe at completion time. Uses data already available at the observation point (CaseDefinition, WorkerOutcome, PlanItemStore).
**Trade-offs:** Delegation evidence is case-level (PlanItemStore query), not per-execution — every worker completion for the same case sees the same delegation state.
**Sources:** WorkflowExecutionCompletedHandler.java:209-214 (success path), WorkflowExecutionCompletedHandler.java:411-412 (failure path)
**Exploration:** quick
**Status:** captured

## D4: Implementation approach

**Choice:** Extend BehavioralComplianceRecorder with recordDelegation and recordEscalation methods alongside existing recordLatency and recordAttestation.
**Alternatives:**
- Separate DelegationObserver and EscalationObserver classes — better separation but breaks single-recorder pattern, adds two new beans and handler call sites.
- Strategy-based ComplianceDimensionObserver SPI — most extensible but over-engineered for 4 dimensions, changes recorder shape significantly.
**Rationale:** Four methods in one recorder is clean. The pattern is proven. Smallest change surface — only the recorder class, its test, and its constructor dependencies change.
**Trade-offs:** Recorder grows from 125 to ~200 lines. If more dimensions are added later, refactoring to a strategy pattern may be warranted.
**Sources:** BehavioralComplianceRecorder.java (existing pattern), BehavioralComplianceRecorderTest.java (existing test pattern)
**Exploration:** quick
**Status:** captured

## D5: Delegation evidence detection

**Choice:** Query PlanItemStore at completion time — check if `PlanItemStore.findByCaseId(caseId, tenancyId)` contains items with `parentCompoundId != null`. Any compound children existing means the case's decomposition structure was exercised. This is case-level evidence, not per-execution.
**Alternatives:**
- EventLog query for SUBCASE_STARTED events — more specific to worker-initiated delegation but requires EventLog query at completion time, crossing module boundaries.
- WorkerRuntime flag — track delegation via a boolean set when `execute()` or `spawnCase()` is called. Per-execution accuracy but requires changes to WorkerRuntime, event threading, and handler.
**Rationale:** PlanItemStore is already available as an SPI. The query is simple and deterministic. Case-level granularity is acceptable for v1 — the TTL + threshold model in eidos absorbs noise from repeated signals.
**Trade-offs:** Every worker completion in the same case records the same delegation state. Pre-execution decomposition (GoalDecomposer at case start) counts as delegation even though the system decomposed, not the agent. Acceptable because goal decomposition is driven by the agent's goal profile.
**Sources:** PlanItemRecord.java:40 (parentCompoundId field), PlanItemStore.java:41 (findByCaseId method)
**Exploration:** quick
**Status:** captured

## D6: Escalation signal semantics

**Choice:** PlannedAction presence = COMPLIANT, WorkerOutcome.Declined = COMPLIANT, no PlannedAction on success = no observation. Two COMPLIANT paths, no VIOLATED path in v1.
**Alternatives:**
- Record VIOLATED when no PlannedAction on any success outcome — too noisy, most tasks don't warrant escalation even for supervised agents.
- Deferred VIOLATED via WorkerRuntime instrumentation — future enhancement, out of scope for v1.
**Rationale:** We can detect escalation behavior (flagging, declining) but cannot detect absence of warranted escalation. Recording only COMPLIANT is honest — it tells the trust system "this agent does escalate" without falsely claiming "this agent fails to escalate." The asymmetry with delegation (which CAN record VIOLATED) is acceptable.
**Trade-offs:** Escalation compliance dimension only contributes positive signals in v1. Agents that never escalate simply have no escalation signals rather than VIOLATED signals. Trust maturity model treats absence of signals as bootstrap phase.
**Sources:** WorkerOutcome.java (Success.plannedAction(), Declined), BehavioralSignal.java (COMPLIANT/VIOLATED)
**Exploration:** quick
**Status:** captured
