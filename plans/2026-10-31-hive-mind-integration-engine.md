# Hive Mind Integration — Engine Issues

**Epic:** #1274 (Hive Mind integration wiring)
**Branch:** `issue-1274-hive-mind-integration`
**Scope:** Engine issues #1275–#1280 (6 issues, 2 batches)

---

## Context

Cross-repo review revealed that engine's swarm mechanics are internally
complete but have thread safety issues and missing integration wiring to
eidos and qhorus. Agency bridge (agency#375) has landed — engine can now
focus on its own house.

## Batch 1: Thread Safety + SPI Cleanup (XS/S issues — #1275, #1276, #1277, #1278)

Independent fixes, no design decisions. Can land in a single commit per
issue or batched.

### Task 1.1 — SignalRegistry thread safety (#1275)

**File:** `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java`

Two changes:
1. Replace `deposit()` get-then-put with `ConcurrentHashMap.compute()`:
   - Current: `existing = caseSignals.get(name)` → conditional → `caseSignals.put(name, ...)`
   - Target: `caseSignals.compute(name, (k, existing) -> { ... })` — atomic read-modify-write
2. Purge expired entries after `markExpired()`:
   - In `findNewlyExpired()`, after marking signals expired, remove them from the map
   - Or add `purgeExpired(UUID caseId)` called from `findNewlyExpired()`

**Test:** Add concurrent deposit test — multiple threads depositing the same signal name, assert reinforcement count is correct.

### Task 1.2 — Swarm state visibility (#1276)

**Files:**
- `runtime-core/.../stigmergy/RoleTracker.java` — `CaseRoleState.currentRoles`, `previousRoles` → `volatile`
- `runtime-core/.../stigmergy/TeamDetector.java` — `CaseTeamState.currentTeams`, `previousTeams` → `volatile`
- `runtime-core/.../stigmergy/SwarmProgressTracker.java` — `CaseProgressState.lastProgress` → `volatile`
- `runtime-core/.../stigmergy/RoleTracker.java` — synchronize `AgentAccumulator` ArrayDeque access in `accumulate()`

**Test:** Existing tests should pass unchanged. Add a focused test that reads state from a different thread than the writer to verify visibility.

### Task 1.3 — SwarmProvisioner listener exception handling (#1277)

**File:** `runtime-core/.../stigmergy/SwarmProvisioner.java`

Wrap each `SwarmCoordinationListener` callback in try-catch:
```java
for (var listener : listeners) {
    try { listener.onSwarmInitialised(...); }
    catch (Exception e) { LOG.warn("Listener failed", e); }
}
```

Apply to: `onSwarmInitialised`, `onProvisioningChanged`, `onRoleChanged`, `onTeamChanged`, `onProgressUpdated`.

**Test:** Add test with a throwing listener + a recording listener — verify the recording listener still fires.

### Task 1.4 — onMemberCompleted wiring (#1278)

**Decision needed:** Wire into `StigmergyCoordinator.agentDeparted()` or remove from SPI.

Recommendation: **wire it** — agent departure is a meaningful swarm event. Agency#375 already has the `MemberCompleted` variant in the sealed hierarchy.

**Files:**
- `api/.../spi/stigmergy/SwarmCoordinationListener.java` — method already declared
- `runtime-core/.../stigmergy/StigmergyCoordinator.java` — add `onMemberCompleted` call in `agentDeparted()`
- `runtime-core/.../stigmergy/SwarmProvisioner.java` — if it delegates departure, wire there instead

**Test:** Verify `onMemberCompleted` fires when an agent departs.

Also bonus: add `SignalDecay` guard for `Duration.ZERO` halfLife — return 0.0 immediately instead of relying on IEEE 754 infinity semantics.

---

## Batch 2: Integration Wiring (M issues — #1279, #1280)

These require design decisions about where and how to integrate. Each
needs brainstorming before implementation.

### Task 2.1 — Wire eidos proximity queries (#1279)

**Integration points to evaluate:**
- `SwarmProvisioner.evaluateAndProvision()` — before provisioning a new agent, query `byProximity()` to check if an existing agent with similar capabilities is available
- `StigmergyCoordinator` — use `coActiveAgents()` to populate neighbor awareness during agent lifecycle events
- `TeamDetector` — enrich team detection with `RuntimeCollaborationQuery.collaborators()` data

**Dependency:** eidos provides these as SPIs. Engine needs to depend on eidos-api (check if dependency already exists) and inject the query beans.

**Design question:** Should engine have a hard dependency on eidos-api, or should it use an optional SPI adapter pattern (engine defines its own discovery SPI, eidos provides the implementation)?

### Task 2.2 — Wire qhorus broadcast (#1280)

**Integration points to evaluate:**
- `SwarmProvisioner` — broadcast `SWARM_PROVISION_COMPLETED` / `SWARM_AGENT_TERMINATED` to `broadcast/<capability>` channels
- `StigmergyCoordinator.detectPatterns()` — broadcast `SIGNAL_CONSENSUS_DETECTED` / `COORDINATION_STORM_DETECTED` to swarm members
- Role/team changes — broadcast role emergence/dissolution to affected agents

**Dependency:** qhorus-api for `MessageDispatcher`. Same dependency question as eidos.

**Design question:** Should broadcasts be fire-and-forget (best effort), or should the swarm wait for delivery confirmation? Fire-and-forget aligns with the stigmergy model (no guaranteed delivery in nature).

---

## Order of Execution

```
Batch 1 (parallel, independent):
  1.1 SignalRegistry (#1275) ─┐
  1.2 Volatile fields (#1276) ├─ can land in any order
  1.3 Listener exceptions (#1277) │
  1.4 onMemberCompleted (#1278) ─┘

Batch 2 (sequential, needs design):
  2.1 Eidos wiring (#1279) ── brainstorm → implement → test
  2.2 Qhorus wiring (#1280) ── brainstorm → implement → test
```

Batch 1 is mechanical — start immediately. Batch 2 needs brainstorming
on the dependency model before implementation.
