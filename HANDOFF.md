# HANDOFF — casehub-engine

## Last Session (2026-10-31)

Epic #1274 — Hive Mind integration wiring. Landed Batch 1 (4 engine issues), completed brainstorming and spec for Batch 2 (2 engine issues).

### Batch 1 — landed on branch `issue-1274-hive-mind-integration-wiring`

- **#1277** — SwarmProvisioner listener exception propagation. Wrapped all 5 listener call sites in try-catch with WARN logging. Added ThrowingCoordinationListener test.
- **#1278** — onMemberCompleted wired into agent departure flow. New `notifyMemberCompleted()` method in SwarmProvisioner, called from `evaluateDeprovisioning()` for each removed agent.
- **#1276** — Swarm state visibility. Added `volatile` to `currentRoles`/`previousRoles` (RoleTracker), `currentTeams`/`previousTeams` (TeamDetector), `lastProgress` (SwarmProgressTracker). Synchronized AgentAccumulator ArrayDeque access in `accumulate()` and `getFingerprint()`.
- **#1275** — SignalRegistry thread safety. Replaced get-then-put race with `ConcurrentHashMap.compute()` in `deposit()`. `markExpired()` now removes expired signals from the map instead of leaving them with `expired=true`.

### Batch 2 — spec written, needs writing-plans then implementation

- **#1279** — Wire eidos proximity queries. Three integration points: SwarmProvisioner (coActiveAgents/topAgentsByOutcome before provisioning), StigmergyCoordinator (neighbor awareness on agentActivated), TeamDetector (RuntimeCollaborationQuery.collaborators enrichment). Direct injection via `Instance<AgentGraphQuery>` / `Instance<RuntimeCollaborationQuery>`.
- **#1280** — Wire qhorus broadcast. Post swarm coordination events to the `coordination` channel via `CaseChannelProvider.postToChannel()` with `MessageType.STATUS` (agent-visible). Support both CDI domain events and channel posting — don't prescribe one strategy. Qhorus-first: channels are observable and participatory.

### Key design decisions (in `specs/issue-1274-hive-mind-integration-wiring/decisions.md`)

- **D1**: Direct injection with `Instance<T>` — no new modules or SPI wrappers
- **D2**: Wire all three eidos integration points
- **D3**: Support both CDI events and coordination channel posting — too early to prescribe
- **D4**: STATUS for agent-visible swarm changes, EVENT for telemetry

### Open implementation detail

- **TenancyId resolution**: eidos/qhorus APIs require tenancyId. Swarm classes currently operate on caseId only. Needs resolution during planning — check if WorkerScope or CaseInstance provides it.

## Immediate Next Step

`work continue` → invoke `writing-plans` with the spec at `specs/issue-1274-hive-mind-integration-wiring/2026-10-31-eidos-qhorus-integration-design.md`. This creates the implementation plan, then execute #1279 and #1280.

## Artifacts Created This Session

| Artifact | Location | Notes |
|----------|----------|-------|
| Spec | `wksp/specs/issue-1274-hive-mind-integration-wiring/2026-10-31-eidos-qhorus-integration-design.md` | Reviewed (light), approved |
| Decisions | `wksp/specs/issue-1274-hive-mind-integration-wiring/decisions.md` | 4 decisions |
| Guidance doc | `parent/docs/platform/messaging-vs-notifications.md` | When to use qhorus vs notifications |
| Protocol | `proj/docs/protocols/casehub/messaging-channel-selection.md` (PP-20261031-060886) | Engine channel selection rule |

## Issues Filed This Session

- **casehubio/platform#682** — Wire unused EngagementType values (REPLIED, CLICKED, CONVERTED) or remove
- **casehubio/qhorus#506** — MessageDispatcher.broadcast() throws UnsupportedOperationException — wire or remove

## Cross-Module

- agency#376 depends on engine#1278 (now landed)
- eidos#213 (enum alignment) can land alongside engine#1279
- GITHUB_TOKEN env var is invalid — blocks `gh` and `git push` unless prefixed with `env -u GITHUB_TOKEN`. Keyring auth (mdproctor) works when GITHUB_TOKEN is unset.
