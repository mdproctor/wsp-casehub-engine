# Decisions — engine#1283 AgentCapabilityGuard

## D1: Full scope — all 5 deferred concerns in one issue

**Choice:** Implement all 5 concerns from the #1282 deferral table in #1283
**Alternatives:**
- Guard + matrix only — focused and testable, but leaves 3 concerns unresolved and needing separate issues
- Guard + matrix + topology validation — partial, still leaves identity type and authority derivation
**Rationale:** All 5 concerns are tightly coupled — the guard needs the authority derivation, which needs the topology validation, which needs to know the identity type. Doing them together avoids incomplete abstractions.
**Trade-offs:** Larger scope means more to review and test. Mitigated by the fact that several concerns are small (identity type = String, topology validation = a switch expression).
**Sources:** specs/issue-1282-coordination-vocabulary-types/SPEC.md §Deferral decomposition
**Exploration:** quick
**Status:** captured

## D2: Permission matrix lives on DecisionScope enum

**Choice:** `DecisionScope.isPermitted(DecisionType)` — scope owns the policy
**Alternatives:**
- In guard implementation — matrix as implementation detail, allows different guards to use different matrices
- Separate PermissionMatrix type — reusable, testable in isolation, but adds a type for a static function
**Rationale:** The matrix is a fixed property of the vocabulary — STIMULUS_DRIVEN always denies TASK_SELECTION regardless of which guard implementation is active. Co-locating with the enum makes this explicit and testable without the guard.
**Trade-offs:** If a future consumer needs a different matrix (unlikely per spec — the matrix is a "total function"), they can't override via SPI. Acceptable — the matrix IS the vocabulary's semantics.
**Sources:** specs/issue-1282-coordination-vocabulary-types/SPEC.md §Permission matrix invariant
**Exploration:** quick
**Status:** captured

## D3: Topology validation and authority derivation on TopologyType enum

**Choice:** `TopologyType.deriveAuthority(StructuralRole)` returns `DecisionAuthority` or throws for invalid combinations
**Alternatives:**
- Separate utility/guard method — keeps enums minimal, puts logic in enforcement layer
**Rationale:** Consistent with D2 — total functions over vocabulary enums live on the governing enum. The derivation is a fixed property of the topology semantics, not a policy choice.
**Trade-offs:** TopologyType gains behavior (a method with a switch expression). Acceptable — it's a pure function with no dependencies.
**Sources:** specs/issue-1282-coordination-vocabulary-types/SPEC.md §Complete derivation mapping
**Exploration:** quick
**Depends on:** D2 (consistent pattern)
**Status:** captured

## D4: DecisionAuthority as a new enum

**Choice:** New enum `DecisionAuthority` with values LOCAL, COORDINATED, CENTRAL in `model.coordination`
**Alternatives:**
- No enum — derive inline without naming the result. Loses expressiveness for downstream consumers (#1284+)
**Rationale:** The #1282 spec removed DecisionAuthority as a standalone *config* enum (to prevent invalid combinations). As a *derived* return type from `TopologyType.deriveAuthority()`, it can't drift — it's always produced by the derivation function, never directly configured.
**Trade-offs:** Adds a type. But it's consumed by downstream SPIs (#1284 TopologyDescriptor, #1286 VotingProtocol) — they need a named type to switch on.
**Sources:** specs/issue-1282-coordination-vocabulary-types/SPEC.md §Resolved Questions #2
**Depends on:** D3 (return type of deriveAuthority)
**Exploration:** quick
**Status:** captured

## D5: Agent identity stays as String

**Choice:** Use `String agentId` — no typed wrapper
**Alternatives:**
- `record AgentId(String value)` — type safety, prevents parameter confusion
**Rationale:** Consistent with existing codebase (WorkerExecutionGuard uses `String workerId`). A typed wrapper would require touching dozens of existing files or living with mixed representations. Can be introduced later if String confusion becomes a real problem.
**Trade-offs:** No compile-time distinction between agent IDs and other Strings. Mitigated by parameter naming and the fact that guard methods have few parameters.
**Sources:** WorkerExecutionGuard.java, OversightGateService.java (both use String)
**Exploration:** quick
**Status:** captured

## D6: AgentAction carries its DecisionType

**Choice:** Each `AgentAction` sealed interface variant carries a `decisionType()` method returning its `DecisionType`
**Alternatives:**
- Guard maps action → type internally — more flexible but the mapping is a fixed property of the action, not a policy choice
**Rationale:** The classification of "DepositSignal is SIGNALING" is intrinsic to the action, not something that varies by guard implementation. Co-locating avoids a mapping table and makes the relationship explicit.
**Trade-offs:** Tight coupling between AgentAction and DecisionType. Acceptable — they were designed together and the mapping is definitional.
**Sources:** specs/issue-1282-coordination-vocabulary-types/SPEC.md §DecisionType
**Exploration:** quick
**Status:** captured

## D7: Guard resolves scope internally

**Choice:** `guard.isPermitted(caseId, agentId, action)` — guard owns scope resolution + matrix check
**Alternatives:**
- Caller resolves scope, passes `(scope, action)` — thinner guard but pushes resolution into every call site
- Both layers public — high-level and low-level API. Unnecessary complexity.
**Rationale:** Follows `WorkerExecutionGuard` precedent (caller passes IDs, guard does the work). Single enforcement point — call sites don't need to know about scope resolution mechanics.
**Trade-offs:** Guard needs access to agent configuration to resolve scope. The NoOp guard avoids this (permits everything). Real implementations will need a scope-resolution mechanism — that's the point of the SPI abstraction.
**Sources:** WorkerExecutionGuard.java (same pattern), ActionRiskClassifier.java (similar orchestrator pattern)
**Exploration:** quick
**Status:** captured
