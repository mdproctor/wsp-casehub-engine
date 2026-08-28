# HumanTask Group Scoring via Group Membership Resolution

**Issue:** engine#757
**Date:** 2026-07-29
**Status:** Draft

## Problem

`HumanTaskRoutingStrategy` scores individual `candidateUsers` but cannot score
`candidateGroups` because group membership resolution is outside the engine's scope.
The `ContextConstraint.Prefer` and `ContextConstraint.Exclude` effects carry `groups`
fields but the constraint strategy ignores them at runtime. CBR scoring only operates
on directly-nominated users.

## Approach

Use the existing `GroupMembershipProvider` SPI from `casehub-platform-api`
(`io.casehub.platform.api.identity`). It provides `membersOf(groupName, tenancyId)
→ Set<GroupMember>` and is already used by `TemplateExpander` in the work repo for
the same purpose. `casehub-platform-api` is already on the engine's dependency tree
(`api/pom.xml`, `runtime/pom.xml`). No new SPI required.

## Design

### 1. Handler expands groups before strategy dispatch

`CaseContextChangedEventHandler.publishHumanTaskSchedule()` injects
`GroupMembershipProvider` and expands all candidate groups at routing time:

```
for each group in resolvedGroups:
    try:
        members = groupMembershipProvider.membersOf(group, tenancyId)
        groupMembership[group] = members.map(GroupMember::actorId)
    catch Exception e:
        log warning "Group expansion failed for {group} — treating as empty"
```

The handler always expands, catching per-group. Failed groups are treated
as having no members — direct candidates and successfully-expanded groups
proceed unaffected. `NoOpGroupMembershipProvider` (`@DefaultBean`,
`casehub-work`) returns empty — zero cost when no real provider is
configured.

### 2. HumanTaskCandidates gains groupMembership

```java
public record HumanTaskCandidates(
    Set<String> groups,
    Set<String> users,
    Map<String, Set<String>> groupMembership) {

    // Compact constructor: null-safe, defensive copies (deep for map)

    public Set<String> allUsers() {
        // union of direct users + all group member actor IDs
    }
}
```

Backward-compatible factory for tests:
```java
public static HumanTaskCandidates of(Set<String> groups, Set<String> users) {
    return new HumanTaskCandidates(groups, users, Map.of());
}
```

### 3. CBR strategy scores allUsers()

`CbrHumanTaskRoutingStrategy.select()` changes:
- Guard: `candidates.allUsers().isEmpty()` (was `candidates.users().isEmpty()`)
- Passes `candidates.allUsers()` as `eligibleWorkerIds` to
  `ExperienceAnalyser.workerSuccessRates()`
- Result: `Enriched(candidates.groups(), candidates.allUsers(), scores)`
  — groups pass through unchanged; candidateUsers includes all
  individually-eligible users (direct + group-expanded); scores cover
  the full eligible set

### 4. Constraint strategy applies group effects

`ConstraintHumanTaskRoutingStrategy.select()` changes:

**Exclude(groups):**
- Resolve group members from `candidates.groupMembership()`
- Remove members from `eligibleUsers` — members are removed regardless
  of whether they were also directly nominated (constraint exclusion is
  a policy override; use user-level `excludeUsers` for targeted
  individual removal that doesn't affect group membership)
- Remove group from `eligibleGroups` (new mutable set, initialized from
  `candidates.groups()`) — otherwise the work repo would override the
  engine's exclusion decision

**Prefer(groups):**
- Resolve group members from `candidates.groupMembership()`
- Collect all users to boost: union of `prefer.users()` and resolved
  group members, deduplicated — prevents double-scoring when a user
  appears in both the user list and a preferred group
- For each boosted user in `eligibleUsers`, merge `constraint.weight()`
  once into their score
- Groups stay in `eligibleGroups` (preference doesn't restrict visibility)

**Workload constraint:**
- Operate on `allUsers()` union — group-expanded users participate in
  workload checks and load-balance scoring

**Eligible users initialization:**
- Start from `candidates.allUsers()` instead of `candidates.users()`

### 4a. Builder fix for mixed group+user effects

`ContextConstraint.Builder` methods `preferGroups()`/`preferUsers()` and
`excludeGroups()`/`excludeUsers()` currently overwrite the `effect` field
rather than accumulating. This is pre-existing but becomes a live bug now
that group effects are functional — `builder.preferGroups(g).preferUsers(u)`
silently discards the groups.

Fix: accumulate within the same effect type. `preferGroups(g)` followed
by `preferUsers(u)` produces `Prefer(g, u)`. Repeated calls to the same
dimension also accumulate: `preferGroups(g1).preferGroups(g2)` produces
`Prefer(g1 ∪ g2, Set.of())`. Switching effect types (e.g.
`preferGroups(g).excludeUsers(u)`) replaces the previous effect
(consistent with current behavior — a constraint has one sealed effect
type).

Additionally, add explicit combined methods:
- `prefer(Set<String> groups, Set<String> users)` → `Prefer(groups, users)`
- `exclude(Set<String> groups, Set<String> users)` → `Exclude(groups, users)`

### 5. Unchanged components

- **HumanTaskScheduleEvent** — no schema change. `resolvedCandidateUsers`
  now carries all individually-eligible users (direct + group-expanded),
  matching `Enriched.candidateUsers()` semantics. Scores for all eligible
  users flow through the existing `candidateScores` map.
  `resolvedCandidateGroups` may have groups removed by constraint exclusion.
- **HumanTaskRoutingResult.Enriched** — same three fields. Invariant:
  `candidateScores.keySet() ⊆ candidateUsers`. `candidateScores` keys are
  individual actor IDs (direct or group-expanded), never group names.
  Javadoc updated to remove the engine#757 caveat and document the
  candidateUsers/candidateScores subset invariant.
- **HumanTaskRoutingContext** — no new fields. Membership is about the
  candidates, not the case context.
- **NoOpHumanTaskRoutingStrategy** — returns `Unchanged`, unaffected.
- **ExperienceAnalyser** — no changes. Already accepts any `Set<String>`
  as eligible IDs.

### 6. Semantic decisions

| Decision | Rationale |
|----------|-----------|
| No group-level scores in `candidateScores` | Work repo assigns to individuals. Group aggregation (avg/max) is a policy decision for the consumer. |
| `Enriched.candidateUsers()` = all individually-eligible users (direct + group-expanded) | Both strategies return the full set of individually-eligible users after processing. Ensures consistent semantics regardless of which strategy produced the result. `candidateScores.keySet() ⊆ candidateUsers` is maintained as an invariant. |
| `Enriched.candidateGroups()` CAN be modified | Constraint Exclude removes groups — otherwise exclusion is meaningless (work repo would still show the task). |
| Handler always expands, catches per-group | NoOp returns empty (zero cost). Expansion failure for one group doesn't block other groups or direct candidates. |
| Membership on candidates, not context | It's data about the candidate groups, not about the case/task. `allUsers()` as a derived method on candidates is natural. |
| Group exclusion overrides direct nomination | Constraints are policies. "Exclude group X" means no members of X regardless of nomination source. Use user-level `excludeUsers` for targeted individual removal. |
| TOCTOU between engine and work repo is accepted | Scores are point-in-time advisory signals for ranking, not gates for eligibility. The work repo resolves groups independently at assignment time and is authoritative for assignment eligibility. Users added to groups after engine resolution have no scores but remain assignable; users removed retain stale scores that are harmless. |

## Testing

- **HumanTaskCandidatesTest** — `allUsers()` union, `groupMembership` defensive copy,
  null-safe construction, backward-compatible `of()` factory
- **CbrHumanTaskRoutingStrategyTest** — scoring includes group-expanded users; empty
  allUsers returns Unchanged; overlap between direct user and group member produces
  single score; candidateUsers in Enriched result contains allUsers()
- **ConstraintHumanTaskRoutingStrategyTest** — Exclude(groups) removes group from
  result and members from eligible; Exclude(groups) removes directly-nominated
  members (policy override); Prefer(groups) boosts member scores; Prefer with
  user in both groups and users applies weight once (dedup); workload constraint
  applies to allUsers; all-excluded-via-groups escalates
- **ContextConstraintBuilderTest** — `preferGroups().preferUsers()` accumulates
  into single `Prefer(groups, users)`; `prefer(groups, users)` creates combined
  effect; switching effect type replaces previous
- **CaseContextChangedEventHandler** — verify `GroupMembershipProvider.membersOf()`
  called per group; verify per-group failure isolation (one group fails, others
  succeed); verify expanded membership threaded to candidates; verify Enriched
  result with group-excluded groups flows to HumanTaskScheduleEvent

## Retention impact

`CbrCaseRetainObserver.buildRoutingKeyMap()` already includes `HumanTaskTarget`
bindings with null `capabilityName`. Plan traces for humanTask bindings carry
`workerName` which is the individual actor who completed the task — group-expanded
users who complete tasks are naturally captured in the CBR case base. No retention
changes needed.
