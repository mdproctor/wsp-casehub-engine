# Handoff — engine#300 + Upstream Rebase
2026-05-21

## What changed this session

**engine#300 closed.** Added `deadline` (ISO-8601 Instant from `PropagationContext`) to
COMMAND content in `WorkerScheduleEventHandler.dispatchCommand()` so claudony can bound
Qhorus Commitment `expiresAt`. Two integration tests, Javadoc on `WorkflowExecutor` /
`WorkerExecutionManager` documenting the `Map<String,Object>` convention, protocol
PP-20260520-981c85, blog entry written and published.

**Upstream rebase.** Pulled 4 treblereel commits (PRs #283, #288) into local main:
`$secret`/`$config` JQ scope variables, schema validation fixes, `WorkItemTemplateService
.findById(UUID)` API change. Three conflicts resolved — all additive. `origin/main` is
now current. Backup at `origin/backup/pre-rebase-upstream-20260521`.

**engine#304 closed.** Test cleanup after rebase: deleted duplicate `templateMode_byName`
test, renamed `templateMode_ambiguousName` → `templateMode_invalidUuidRef`.

**Tracked for later:** engine#301 (typed `CommandContent` record), engine#302
(`CaseHub.startCase(Object)` alignment with `Flow.instance(Object)`), engine#303
(provisioner test timing flakiness).

**All 15 previously-unpushed commits** from prior session are now on `origin/main`.

## Immediate Next Step

Pick up #274: BlackboardRegistry hydration from PlanItemStore on restart.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| #302 | Align `CaseHub.startCase` to `Object` (quarkus-flow parity) | M | Med | Design needed first |
| #301 | Typed `CommandContent` record replacing raw Map | S | Low | After #302 |
| #253 | Assess quarkus-hibernate-reactive-panache compile-scope dep | S | Med | — |
| #254 | Java 21 platform migration | L | Med | — |
| #277 | json-schema-validator version conflict in work-adapter | XS | Low | — |
| work#174 | DB-level UNIQUE on WorkItemTemplate.name | S | Low | — |
| work#175 | JSON merge semantics defaultPayload + inputData | S | Med | — |
| Devtown | Epic 3: CasePlanModel PR review | M | Med | Queued multiple sessions |

## Key references

- Blog: `blog/2026-05-20-mdp02-the-deadline-gets-through.md`
- Garden: GE-20260520-c0e5b4 (Podman DOCKER_HOST gotcha)
- Protocol: PP-20260520-981c85 (inputData Map convention)
