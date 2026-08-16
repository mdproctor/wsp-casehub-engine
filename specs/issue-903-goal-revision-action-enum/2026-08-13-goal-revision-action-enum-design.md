# GoalRevisionAction Enum — Type-Safe Goal Lifecycle Transitions

**Issue:** engine#903
**Depends on:** engine#806 (goal revision — landed)

## Problem

`GoalRevisionProposal.RevisedGoal` has three fields: `goalName`,
`revisedDescription` (nullable), and `revisionReason`. This models only one
lifecycle transition: description revision.

Goals have three distinct transitions after formation:

1. **Revision** — the goal description should be refined based on outcome data
2. **Abandonment** — the goal is dropped (unachievable or no longer relevant)
3. **Completion** — the goal succeeded (positive signal)

Without type-safe representation, consumers resort to string conventions in
`revisionReason` (e.g., "ABANDON:" prefix). This is fragile, undiscoverable,
and semantically overloaded.

## Design

Add a `GoalRevisionAction` enum and an `action` field to `RevisedGoal`.
Update `GoalRevisionEvaluator` to handle all three actions. Update
`LlmGoalRevisionStrategy` prompt and parsing to produce the new actions.

### Scope distinction

`GoalRevisionAction.COMPLETE` is an **agent-level** concept — removing a goal
from the agent descriptor after accumulated evidence across many cases shows
the goal is permanently achieved.

`AgentGoalCompletionMarker` (engine#785) is a **case-level** concept — marking
`_agentGoals.<agentId>.<goalName>.met = true` per-case-execution for
GoalExpression evaluation. These operate at different scopes with no
coordination needed. After agent-level completion removes a goal from the
descriptor, the case-level marker stops firing (no goal in descriptor →
`markGoalsCompleted` returns early).

## Components

### GoalRevisionAction (new — engine-api)

`io.casehub.api.spi.routing.GoalRevisionAction` — top-level enum.

```java
public enum GoalRevisionAction {
    REVISE,
    ABANDON,
    COMPLETE
}
```

### RevisedGoal (modified — engine-api)

Add `action` as the second record component. No backward-compatible
constructor — all callers must specify the action explicitly (1 production
caller + tests, mechanical migration).

```java
public record RevisedGoal(
    String goalName,
    GoalRevisionAction action,
    String revisedDescription,
    String revisionReason
) {
    public RevisedGoal {
        Objects.requireNonNull(goalName, "goalName must not be null");
        Objects.requireNonNull(action, "action must not be null");
        Objects.requireNonNull(revisionReason, "revisionReason must not be null");
        if (action == GoalRevisionAction.REVISE && revisedDescription == null) {
            throw new IllegalArgumentException(
                "revisedDescription is required for REVISE action");
        }
    }
}
```

Validation: `action` is non-null always. `revisedDescription` required for
REVISE (fail-fast), nullable for ABANDON/COMPLETE.

### GoalRevisionEvaluator (modified — runtime)

Rename `mergeDescriptions()` to `applyRevisions()`. Switch on `action` per
revision in a single pass:

```java
private List<AgentGoal> applyRevisions(
    List<AgentGoal> goals, GoalRevisionProposal proposal) {
    Map<String, GoalRevisionProposal.RevisedGoal> revisionsByGoal = new HashMap<>();
    for (var revision : proposal.revisions()) {
        revisionsByGoal.put(revision.goalName(), revision);
    }
    if (revisionsByGoal.isEmpty()) {
        return goals;
    }

    List<AgentGoal> result = new ArrayList<>();
    for (AgentGoal goal : goals) {
        var revision = revisionsByGoal.get(goal.name());
        if (revision == null) {
            result.add(goal);
            continue;
        }
        switch (revision.action()) {
            case REVISE -> {
                try {
                    result.add(goal.toBuilder()
                        .description(revision.revisedDescription()).build());
                } catch (Exception e) {
                    LOG.warnf("Invalid description for goal %s, keeping original: %s",
                        goal.name(), e.getMessage());
                    result.add(goal);
                }
            }
            case ABANDON, COMPLETE -> {
                // Goal excluded from result — removed from descriptor
            }
        }
    }
    return result;
}
```

Contract change: the method can now return a smaller list than the input.
`handleEvolved()` already handles varying goal counts (GoalEvolution can
change list size). The rename from `mergeDescriptions` to `applyRevisions`
signals the broader responsibility.

### Audit log enrichment (modified — runtime)

`handleEvolved()` extracts per-action goal lists from the proposal
**before** calling `applyRevisions()`, then passes them to
`writeAuditLog()` via an updated signature:

```java
private void writeAuditLog(
    String agentId, String tenancyId,
    GoalEvolutionResult.Evolved evolved,
    Map<String, GoalOutcomeCounts> counts,
    List<String> abandonedGoals,
    List<String> completedGoals) { ... }
```

New audit metadata fields:

| Key | Type | Description |
|-----|------|-------------|
| `abandonedGoals` | `List<String>` | Goal names with ABANDON action |
| `completedGoals` | `List<String>` | Goal names with COMPLETE action |
| `descriptionRevisions` | `List<Object>` | Goals with REVISE action and changed descriptions |

These are alongside existing `promotedGoals` and `demotedGoals` from
GoalEvolution.

### LlmGoalRevisionStrategy (modified — runtime)

**Prompt update:** Expand the system prompt to include abandonment and
completion in the LLM's vocabulary:

```
You are a goal effectiveness analyst. Given an agent's goals and their
performance metrics, evaluate each goal and recommend one of three actions:
- REVISE: refine the goal description to better capture what the agent
  accomplishes. Only when meaningfully misaligned with observed outcomes.
- ABANDON: drop the goal entirely. Only when the goal is unachievable or
  no longer relevant based on persistent failure patterns.
- COMPLETE: mark the goal as achieved. Only when the goal has been
  consistently met and keeping it adds no further value.
Respond with JSON only.
```

**JSON schema update:** Add `action` field to the response schema:

```json
{
  "revisions": [{
    "goalName": "...",
    "action": "REVISE|ABANDON|COMPLETE",
    "revisedDescription": "...",
    "revisionReason": "..."
  }],
  "rationale": "..."
}
```

**Parsing update:** `parseResponse()` reads the `action` field with two
tolerance rules for LLM output:
- Absent `action` field → defaults to `REVISE`
- Invalid `action` string (e.g., "REMOVE", "drop") → defaults to `REVISE`

The enum on `RevisedGoal` itself has no default — the parser is the
tolerance boundary. Invalid values do not discard the revision entry or
fail the entire proposal.

## Module Placement

| Type | Module | Rationale |
|------|--------|-----------|
| `GoalRevisionAction` | engine-api | Proposal-level concept in revision SPI |
| `RevisedGoal` (modified) | engine-api | Existing location |
| `GoalRevisionEvaluator` (modified) | runtime | Existing location |
| `LlmGoalRevisionStrategy` (modified) | runtime | Existing location |

No cross-repo changes. No new modules.

## Testing

### Unit tests

1. **`GoalRevisionProposalTest`** (modified):
   - REVISE action with valid description accepted
   - REVISE action with null description throws `IllegalArgumentException`
   - ABANDON action with null description accepted
   - COMPLETE action with null description accepted
   - ABANDON/COMPLETE action with non-null description accepted (informational)
   - Null action throws `NullPointerException`
   - Existing tests updated to pass `GoalRevisionAction.REVISE` explicitly

2. **`GoalRevisionEvaluatorTest`** (modified):
   - REVISE action: existing behavior unchanged (description updated)
   - ABANDON action: goal excluded from returned list
   - COMPLETE action: goal excluded from returned list
   - Mixed actions: REVISE + ABANDON + COMPLETE in same proposal
   - Unknown goal name with ABANDON/COMPLETE: silently ignored
   - REVISE with invalid description: per-goal error isolation, goal kept
   - Audit log contains `abandonedGoals` and `completedGoals` lists

3. **`LlmGoalRevisionStrategyTest`** (modified):
   - Parses REVISE action from JSON
   - Parses ABANDON action from JSON
   - Parses COMPLETE action from JSON
   - Missing `action` field defaults to REVISE in parser
   - System prompt includes abandonment/completion vocabulary

## Scope Boundaries

**In scope:**
- `GoalRevisionAction` enum (new)
- `RevisedGoal` — add `action` field, validation per action
- `GoalRevisionEvaluator.applyRevisions()` — rename + action handling
- `GoalRevisionEvaluator.writeAuditLog()` — enriched metadata
- `LlmGoalRevisionStrategy` — prompt, schema, and parsing updates
- Tests updated for all changes

**Out of scope:**
- `GoalAbandonmentEvaluator` — remains as a complementary routing-time
  soft filter (threshold-based, temporary). Not replaced by ABANDON action.
- `AgentGoalCompletionMarker` — case-level concept, no changes needed
- `GoalEvolutionResult` in eidos-api — structural decisions stay in eidos
- `GoalFormationEvaluator` — no changes, but benefits from freed capacity
  when goals are removed
- `GoalSignalStore.clear()` — existing behavior, no changes
- `GoalSignalProvider` routing score — `activeGoals.size() / totalGoals.size()`
  naturally adjusts when goals are removed (correct: agent is fully engaged
  with remaining goals)

## Review Findings Addressed

| Finding | Source | Resolution |
|---------|--------|------------|
| Backward-compat constructor defeats enum purpose | R1-02 | Accepted: dropped 3-arg constructor |
| Dead code without prompt update | R1-08 | Accepted: LLM prompt and parsing changes included |
| Sealed interface reasoning | R1-03 | Noted: same data shape + platform pattern (GoalPriority) favours enum |
| Enum in eidos-api | R1-04 | Rejected: proposal-level concept, not domain lifecycle status |
| ABANDON/COMPLETE conflates transitions | R1-06 | Rejected: case-level vs agent-level distinction; signal clearing is existing behavior |
| LLM gains structural authority | R1-07 | Rejected: LLM already exercises this via string conventions; change makes implicit authority type-safe |
| Method contract change | R1-09 | Noted: rename signals the change; handleEvolved() handles varying sizes |
| writeAuditLog doesn't receive proposal | Spec R1-04 | Accepted: specified data flow from proposal extraction to updated signature |
| Invalid LLM action values unhandled | Spec R1-05 | Accepted: invalid action strings default to REVISE (parser tolerance) |
| GoalSignalProvider routing impact | Spec R1-07 | Noted: acknowledged in scope boundaries |
