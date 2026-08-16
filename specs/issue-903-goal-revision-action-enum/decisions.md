## D1: Enum design and backward compatibility

**Choice:** Enum field, no backward-compat constructor — `GoalRevisionAction` as a top-level enum in `io.casehub.api.spi.routing`, `action` as second field on `RevisedGoal`. All callers must specify the action explicitly (1 production caller + tests — mechanical migration).
**Alternatives:**
- Sealed interface hierarchy — same data shape with discriminator-only difference favours enum (matches `GoalPriority`-on-`AgentGoal` platform pattern); sealed interfaces suit genuinely polymorphic types like `GoalEvolutionResult`
- String-typed action field — loses compile-time safety, explicitly rejected in the issue as fragile and undiscoverable
- Backward-compatible 3-arg constructor defaulting to REVISE — rejected per review R1-02: silent default defeats the enum's purpose on a 1-caller API
**Rationale:** Minimal change with type safety and switch exhaustiveness. Breaking the constructor forces every caller to declare intent — which is the point of adding the enum.
**Trade-offs:** `revisedDescription` nullability is convention-enforced per action (validated in compact constructor) rather than structurally prevented
**Exploration:** quick
**Status:** revised (R1-02 accepted: dropped backward-compat constructor)

## D2: How mergeDescriptions() handles ABANDON and COMPLETE

**Choice:** Rename to `applyRevisions()`, filter by action in a single pass — REVISE updates description, ABANDON/COMPLETE exclude the goal from the returned list. Semantic difference captured in audit metadata only.
**Alternatives:**
- Separate passes (descriptions first, removals second) — unnecessary complexity for three cases
- Separate handler methods per action — YAGNI, three cases in a switch is the right abstraction
**Rationale:** Single pass with clean switch, rename reflects broader responsibility beyond description merging
**Trade-offs:** ABANDON and COMPLETE have identical structural effect (removal) — differentiation is audit-only in v1
**Exploration:** quick
**Status:** captured
