## D1: Delegation eligibility policy (Gate 2)

**Choice:** Structural signals as Gate 2 — after the eidos disposition gate passes (`BehavioralExpectations.delegationExpected(disposition)` = Gate 1), the engine checks whether delegation is structurally possible for this case: `CaseDefinition.getDecompositionStrategy() != null` OR any binding has a `SubCaseTarget`.
**Alternatives:**
- Outcome-only — observe whether agent delegated without assessing if warranted. Simple but noisy — agents correctly handling simple tasks get VIOLATED.
- Capability complexity — use multi-capability binding or sub-capability count. Vague, requires additional metadata.
**Rationale:** Follows the existing two-gate pattern. Gate 1 (eidos) determines if the agent is expected to delegate in general. Gate 2 (engine) determines if this specific case has decomposition infrastructure. Both gates must pass before observation occurs.
**Trade-offs:** Case definitions without decomposition infrastructure get no delegation observation, even if delegation would have been appropriate in principle.
**Sources:** BehavioralComplianceRecorder.java:80-110 (latency gate pattern), ComplianceDimension.java (DELEGATION constant), BehavioralExpectations.delegationExpected() (Gate 1)
**Exploration:** quick
**Status:** revised (R1-02 clarification — Gate 1/Gate 2 framing made explicit)

## D2: Escalation eligibility policy (Gate 2)

**Choice:** No additional Gate 2 for escalation eligibility. Gate 1 alone (`BehavioralExpectations.escalationExpected(descriptor, registry)`) determines eligibility — it checks whether the agent's autonomy vocabulary implies supervision. Every completion by a supervised agent is observed.
**Alternatives:**
- PlannedAction presence as Gate 2 — only observe when the agent flagged a consequential action. But this prevents VIOLATED signals (can't detect when agent SHOULD have escalated but didn't).
- Risk classifier registered — system-level gate. Doesn't help with per-agent observation.
**Rationale:** The eidos gate is already narrow — only agents whose autonomy vocabulary `impliesSupervision()` pass. This is a small subset of agents. Observing all completions for this subset enables both COMPLIANT and VIOLATED signals.
**Trade-offs:** Supervised agents that correctly handle routine tasks autonomously receive VIOLATED signals. The TTL + threshold model absorbs these — occasional autonomous completions don't trigger probes; persistent autonomous behavior does.
**Sources:** BehavioralExpectations.escalationExpected() (Gate 1), VocabularyTerm.impliesSupervision(), AgentDisposition.autonomy()
**Exploration:** quick
**Status:** revised (R1-03, R1-05 — separated eligibility from signal detection, enabled VIOLATED signals)

## D3: Observation point

**Choice:** Single observation point at worker completion — keep all recording in BehavioralComplianceRecorder, called from WorkflowExecutionCompletedHandler on both success and failure paths.
**Alternatives:**
- Multi-point — add hooks in WorkerRuntime (delegation detection), ActionGate pipeline (escalation detection). Richer signals but touches multiple modules and breaks the single-recorder pattern.
**Rationale:** Consistent with LATENCY and ATTESTATION_RATE dimensions which both observe at completion time. Uses data already available at the observation point (CaseDefinition, WorkerOutcome, PlanItemStore).
**Trade-offs:** Delegation evidence is case-level (PlanItemStore query), not per-execution — every worker completion for the same case sees the same delegation state.
**New dependencies:** `Instance<PlanItemStore>` (delegation evidence query — blocking I/O, acceptable on `@RunOnVirtualThread` handler context) and `VocabularyRegistry` (escalation eligibility — already `@ApplicationScoped` via `NoOpVocabularyRegistry` or `CdiVocabularyRegistry`).
**Sources:** WorkflowExecutionCompletedHandler.java:209-214 (success path), WorkflowExecutionCompletedHandler.java:411-412 (failure path)
**Exploration:** quick
**Status:** revised (R1-06 — documented new constructor dependencies and blocking query concern)

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
**Rationale:** PlanItemStore is already available as an SPI. The query is simple and deterministic.
**Trade-offs:** Every worker completion in the same case records the same delegation state. Pre-execution decomposition (GoalDecomposer at case start) counts as delegation even though the system decomposed, not the agent. **Known limitation for v1:** this measures case topology, not per-agent behavior. An agent that doesn't delegate but executes in a pre-decomposed case receives COMPLIANT signals it didn't earn. Acceptable for v1 because (a) most decomposed cases are designed for delegating agents, (b) the TTL+threshold model absorbs noise, and (c) per-execution tracking (WorkerRuntime flag) is the planned v2 enhancement.
**Sources:** PlanItemRecord.java:40 (parentCompoundId field), PlanItemStore.java:41 (findByCaseId method)
**Exploration:** quick
**Status:** revised (R1-04 — explicitly documented case-level measurement limitation and v2 path)

## D6: Escalation signal semantics

**Choice:** Three signal paths for supervised agents (Gate 1 passed):
- `WorkerOutcome.Success` with `PlannedAction` present → COMPLIANT (agent flagged for oversight)
- `WorkerOutcome.Declined` → COMPLIANT (agent acknowledged it can't handle this — a form of escalation)
- `WorkerOutcome.Success` without `PlannedAction` → VIOLATED (supervised agent acted autonomously)
- `WorkerOutcome.Failed` / `WorkerOutcome.Expired` → no escalation observation (failure is not an escalation decision)
**Alternatives:**
- COMPLIANT-only (original D6) — only record when escalation behavior is detected. Safer but prevents the trust system from learning "this agent fails to escalate."
**Rationale:** The eidos gate already narrows to supervised agents only. For supervised agents, autonomous completion IS a meaningful negative signal — the agent was expected to operate under oversight but didn't flag. The trust model needs both positive (does escalate) and negative (doesn't escalate) signals to assess escalation compliance.
**Trade-offs:** Supervised agents correctly handling routine tasks autonomously get VIOLATED. The threshold model absorbs occasional autonomous completions — persistent autonomous behavior triggers probes. **Cross-dimension interaction (deliberate):** `WorkerOutcome.Declined` records VIOLATED for ATTESTATION_RATE and COMPLIANT for ESCALATION simultaneously. This is semantically correct: declining hurts reliability but demonstrates safety judgment. The net effect on trust depends on dimension weights configured per deployment.
**Sources:** WorkerOutcome.java (Success.plannedAction(), Declined), BehavioralSignal.java (COMPLIANT/VIOLATED), R1-05, R1-07
**Exploration:** quick
**Status:** revised (R1-05 — enabled VIOLATED signals; R1-07 — documented Declined cross-dimension interaction)
