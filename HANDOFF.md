# HANDOFF — casehub-engine (Slot 160)

## Session Summary (2026-08-30)

Governed yield reconciliation (#994). The v2 branch was abandoned because main received parallel work (#995, #999, #1000) that partially covered it. This session filed and executed 5 focused issues for the first round of gaps, then did a second gap analysis that revealed the v2 branch had significantly richer types than what we ported.

## Phase 1 — Completed (5 issues, all closed)

| Issue | Repo | SHA | What |
|-------|------|-----|------|
| engine#1009 | engine | `db23e878` | CallerConfig sealed (Human/Llm/A2A/Any), typed Evidence, CallerIdentity, enriched VerificationContext/EscalationContext |
| engine#1010 | engine | `c45ab268` | JudgmentPayload sealed (BindingPayload/GatePayload), JudgmentRequest, CloudEventJudgmentScheduler, CallerRefParser.JudgmentRef |
| engine#1011 | engine | `cf3c03ea` | Escalate(CallerConfig, reason) replaces RouteHigher, DefaultJudgmentEscalator heuristic |
| blocks#219 | blocks | `55a11b9` | JudgmentPhase SPI, JudgmentContext, JudgmentDecision sealed, ExecutionModel.judgment field |
| qhorus#422 | qhorus | `f7753a47` | MessageType.JUDGMENT — commissive speech act for governed yield |

All landed on fork main (not upstream). Repos in the slot are on main at origin/main.

## Phase 2 — Remaining (4 issues, filed but not started)

Second gap analysis revealed the v2 branch had significantly richer types. The foundation types from Phase 1 are correct but too thin for real usage.

### Execution sequence (dependency order):

**1. engine#1012 — Enrich CallerConfig, CallerIdentity, Evidence** (M / Med)
   - `CallerConfig.Human`: 2 fields → 12 (add CandidateSetSpec groups/users, title, titleExpression, outcomes, claimDeadlineHours, scope, scopeExpression, priority, templateRef, payloadType, quorum)
   - `CallerConfig.Llm`: 1 field → 3 (add `modelName`, `systemPrompt`)
   - `CallerConfig.A2A`: add `streaming` boolean
   - `CallerIdentity`: add `trustScore` (Double, nullable)
   - `Evidence`: add `ref` (nullable String, external reference)
   - `JudgmentTarget`: add per-target `maxEscalationAttempts`
   - Add `@Deprecated(forRemoval=true)` on `HumanTaskTarget`, `HumanTaskScheduler`
   - Replace `DelegatingJudgmentScheduler` with clean `NoOpJudgmentScheduler` `@DefaultBean`
   - **Approach:** cherry-pick from v2 where possible, re-implement against current API
   - **Blocked by:** nothing

**2. blocks#220 — Wire judgment loop in AbstractExecutionDriver** (S / Low)
   - Add Phase 3.5 judgment integration in the execution loop (after aggregation, before termination)
   - `JudgmentContext` construction with `lastJudgmentFeedback` threading
   - `Rejected` → re-iterate with feedback, `Escalated` → break, `Approved` → continue
   - Add `ExecutionEventListener.onJudgment(JudgmentDecision)` default method
   - Cherry-pick tests: `ExecutionModelJudgmentTest` (175 lines), `SupervisorJudgmentTest` (96 lines)
   - **Blocked by:** engine#1012 (CallerConfig.Llm.systemPrompt for test fixtures)

**3. blocks#221 — Port engine-adapter judgment types** (M / Med)
   - `PatternJudgmentConfig` — record with prompt, callerConfig, verifierStrategy, evidenceRequirements, mode
   - `LlmJudgmentPhase` — ChatModel-based judgment evaluator with APPROVE/REJECT parsing
   - `LlmJudgmentScheduler` — bridges pattern judgment to engine's JudgmentScheduler
   - `SchemaValidationVerifier`, `LlmEvaluationVerifier` — verification strategies
   - YAML parsing in `PatternWorkerFunctionProvider`
   - Handler integration in `PatternWorkerFunctionHandler`
   - Cherry-pick tests: `LlmJudgmentPhaseTest`, `LlmJudgmentSchedulerTest`, `PatternJudgmentYamlTest`, `SchemaValidationVerifierTest`
   - **Blocked by:** engine#1012, blocks#220

**4. engine#1013 — DagNode.judgment field** (M / Med)
   - Nullable `JudgmentTarget judgment` on `DagNode` for yield gates in DAG execution
   - `DagNodeSnapshot.hasJudgment` for REST visibility
   - DagDriver JUDGING state integration
   - **Blocked by:** engine#1012, blocks#221

## Slot-Local M2 Issues

The slot-local `.m2` had stale SNAPSHOTs that were fixed this session:
- `casehub-worker-api` — copied from global m2 (old `inputSchema`/`outputSchema` → new `inputProjection`/`outputProjection`)
- `casehub-neocortex-cognitive-api` — was missing entirely, copied from global m2
- `casehub-neocortex-parent` — old POM lacked cognitive-api in dependencyManagement

**Pre-existing test compilation breaks (NOT fixed — out of scope):**
- engine: `AgentExperienceRecorderTest` references `Outcome.importance()` — stale neocortex SNAPSHOT
- blocks: `StrategyLearningOrchestrator` references `EngagementEvent.sentimentShift()` — same cause
- These block `mvn test` on the runtime/blocks modules. Workaround: temporarily move broken file to `/tmp/`, run specific tests, restore

## V2 Branch Reference

The abandoned branches contain the source material for cherry-picking:
- **engine:** `issue-994-governed-yield-v2` (9 commits) — CallerConfig with full fields, Evidence with ref, JudgmentTarget.maxEscalationAttempts, DagNode.judgment, CLAUDE.md updates
- **blocks:** `issue-994-governed-yield` (5 judgment commits at tip) — JudgmentPhase/Context/Decision, ExecutionModel.judgment, LlmJudgmentPhase, PatternJudgmentConfig, YAML parsing, handler wiring, 4 test classes
- **qhorus:** `issue-994-governed-yield` (1 judgment commit) — already cherry-picked as #422

**Do not delete these branches** — they're the cherry-pick source for Phase 2.

## What's Next

Start with engine#1012 (enrich foundation types). It unblocks everything else. Approach: diff v2's CallerConfig/CallerIdentity/Evidence against current main, cherry-pick the field additions, adapt to current imports. This is a single-session M-scale issue.

After #1012: blocks#220 (S, driver wiring — cherry-pick + import fix), then blocks#221 (M, engine-adapter types — cherry-pick + adapt), then engine#1013 (M, DagNode.judgment — cherry-pick + DagDriver integration).
