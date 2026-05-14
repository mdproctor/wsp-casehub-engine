# Handoff — Sub-case M-of-N and Concurrency
2026-05-13

## What changed this session

**engine#112 closed — sub-case M-of-N coordination complete.**

12 commits to `casehub-engine` main. 157 blackboard tests green.

Key additions:
- `SubCase` gains `groupId`, `totalInGroup`, `requiredCount`, `onThresholdReached`
- `CaseInstance.parentCaseId` — child knows its parent
- `PropagationContext.createChild()` propagated via new 4-arg `CaseHubRuntime.startCase` overload
- `SubCaseGroup` POJO + `SubCaseGroupRepository` SPI (memory + JPA implementations)
- `SubCaseGroupPolicy` — pure static M-of-N threshold arithmetic
- `SubCaseExecutionHandler` — grouped path; writes `groupId` into SUBCASE_STARTED EventLog metadata
- `SubCaseCompletionListener` — splits grouped/ungrouped; atomic `markPolicyTriggered` returns `Uni<Boolean>` (whether THIS call won the CAS); REJECTED cancels parent

**Race condition fixed in code review** — two concurrent child completions could both see COMPLETED before either set `policyTriggered`. Fix: `markPolicyTriggered` returns `Uni<Boolean>`; JPA uses conditional `UPDATE WHERE policyTriggered = false`; memory uses `synchronized` CAS.

**Three follow-up issues opened:**
- engine#248 — JPA pessimistic locking (conditional UPDATE correct but not bulletproof under extreme concurrency)
- engine#249 — Fire `SubCaseGroupLifecycleEvent` via CDI for monitoring consumers
- engine#252 — `SubCaseCompletionListener` refactor (see below)

**engine#252 — `SubCaseCompletionService` extraction:**
`@ObservesAsync` events never fire in `@QuarkusTest` — current tests call listener directly as a workaround. Real fix: extract coordination logic into `SubCaseCompletionService` with constructor injection (works as CDI bean OR plain Java). Listener becomes a 5-line delegator. No scope annotation = `@Dependent`; add `@ApplicationScoped` only if state is needed.

**Protocol added:** `subcase-coordination-strategy.md` (parent/docs/protocols/) — native M-of-N for counting, quarkus-flow for conditional/sequential orchestration, always behind SPI.

**CLAUDE.md updated** — `SubCaseGroup`/`SubCaseGroupRepository` in Persistence Architecture; `@ObservesAsync` test limitation in casehub-blackboard section.

## Immediate next actions

1. **engine#252** — `SubCaseCompletionService` extraction (constructor injection, thin listener, clean tests)
2. **clinical#3** — now unblocked; multi-site sub-case orchestration can proceed
3. **Devtown Epic 3** — PR review CasePlanModel (queued before this session)
4. **parent#13** — Claude config restructuring (still open)

## Key references

- Spec: `specs/2026-05-12-subcase-mofn-coordination-design.md`
- Plan: `plans/2026-05-12-subcase-mofn-implementation.md`
- Blog: `blog/2026-05-13-mdp01-subcase-coordination-concurrency.md`
- Protocol: `~/claude/casehub/parent/docs/protocols/subcase-coordination-strategy.md`
- Garden: `GE-20260513-b15933` — `@ObservesAsync` not delivered in `@QuarkusTest`
