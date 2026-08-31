# HumanTask Group Scoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #757 — HumanTask group scoring via group membership resolution
**Issue group:** #754, #755, #757, #756

**Goal:** Enable HumanTask routing strategies to score group-expanded users
by integrating the existing `GroupMembershipProvider` SPI from `casehub-platform-api`.

**Architecture:** The handler (`CaseContextChangedEventHandler`) expands
candidate groups to members via `GroupMembershipProvider.membersOf()` before
dispatching to the routing strategy. `HumanTaskCandidates` gains a
`groupMembership` map and `allUsers()` derived method. Both CBR and
Constraint strategies use the expanded user set for scoring and effects.

**Tech Stack:** Java 21, Quarkus, CDI, `casehub-platform-api` (`GroupMembershipProvider`)

## Global Constraints

- `GroupMembershipProvider` is from `io.casehub.platform.api.identity` (already on classpath)
- `MockGroupMembershipProvider` (`@DefaultBean`) returns empty — zero cost when unconfigured
- `candidateScores` keys are individual actor IDs only, never group names
- `Enriched.candidateUsers()` = all individually-eligible users (direct + group-expanded)
- Invariant: `candidateScores.keySet() ⊆ candidateUsers`
- Group exclusion overrides direct nomination (policy semantics)
- TOCTOU between engine and work repo is accepted (scores are advisory)
- Use `ide_*` tools for all Java edits; `ide_build_project` for verification

---

### Task 1: HumanTaskCandidates gains groupMembership and allUsers()

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/HumanTaskCandidates.java`
- Modify: `api/src/test/java/io/casehub/api/spi/routing/HumanTaskCandidatesTest.java`

**Interfaces:**
- Produces: `HumanTaskCandidates(Set<String> groups, Set<String> users, Map<String, Set<String>> groupMembership)` record constructor
- Produces: `Set<String> allUsers()` — union of `users` + all `groupMembership` values
- Produces: `static HumanTaskCandidates of(Set<String> groups, Set<String> users)` — backward-compat factory returning `new HumanTaskCandidates(groups, users, Map.of())`

- [ ] **Step 1: Write failing tests for new groupMembership field and allUsers()**

```java
@Test
void allUsersUnionOfDirectAndGroupExpanded() {
  var c = new HumanTaskCandidates(
      Set.of("managers"), Set.of("alice"),
      Map.of("managers", Set.of("bob", "charlie")));
  assertThat(c.allUsers()).containsExactlyInAnyOrder("alice", "bob", "charlie");
}

@Test
void allUsersDeduplicatesOverlap() {
  var c = new HumanTaskCandidates(
      Set.of("managers"), Set.of("alice"),
      Map.of("managers", Set.of("alice", "bob")));
  assertThat(c.allUsers()).containsExactlyInAnyOrder("alice", "bob");
}

@Test
void allUsersFallsBackToUsersWhenNoMembership() {
  var c = new HumanTaskCandidates(Set.of("managers"), Set.of("alice"), Map.of());
  assertThat(c.allUsers()).containsExactly("alice");
}

@Test
void groupMembershipDefensiveCopy() {
  var members = new java.util.HashMap<String, Set<String>>();
  members.put("g", Set.of("u1"));
  var c = new HumanTaskCandidates(Set.of("g"), Set.of(), members);
  members.put("g2", Set.of("u2"));
  assertThat(c.groupMembership()).doesNotContainKey("g2");
}

@Test
void nullGroupMembershipDefaultsToEmpty() {
  var c = new HumanTaskCandidates(Set.of(), Set.of("alice"), null);
  assertThat(c.groupMembership()).isEmpty();
  assertThat(c.allUsers()).containsExactly("alice");
}

@Test
void ofFactoryCreatesEmptyMembership() {
  var c = HumanTaskCandidates.of(Set.of("g"), Set.of("u"));
  assertThat(c.groups()).containsExactly("g");
  assertThat(c.users()).containsExactly("u");
  assertThat(c.groupMembership()).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=HumanTaskCandidatesTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure (constructor signature changed)

- [ ] **Step 3: Implement HumanTaskCandidates changes**

Replace the entire record with `ide_edit_member` (member=`HumanTaskCandidates`):

```java
public record HumanTaskCandidates(
    Set<String> groups, Set<String> users, Map<String, Set<String>> groupMembership) {
  public HumanTaskCandidates {
    groups = groups != null ? Set.copyOf(groups) : Set.of();
    users = users != null ? Set.copyOf(users) : Set.of();
    groupMembership =
        groupMembership != null
            ? groupMembership.entrySet().stream()
                .collect(
                    java.util.stream.Collectors.toUnmodifiableMap(
                        Map.Entry::getKey, e -> Set.copyOf(e.getValue())))
            : Map.of();
  }

  public Set<String> allUsers() {
    if (groupMembership.isEmpty()) {
      return users;
    }
    var all = new java.util.LinkedHashSet<>(users);
    groupMembership.values().forEach(all::addAll);
    return Set.copyOf(all);
  }

  public static HumanTaskCandidates of(Set<String> groups, Set<String> users) {
    return new HumanTaskCandidates(groups, users, Map.of());
  }
}
```

- [ ] **Step 4: Fix compilation in existing call sites**

Update `CaseContextChangedEventHandler.publishHumanTaskSchedule()` (line 575):
Change `new HumanTaskCandidates(resolvedGroups, resolvedUsers)` to
`HumanTaskCandidates.of(resolvedGroups, resolvedUsers)`.

Update `CbrHumanTaskRoutingStrategyTest.candidates()` helper:
Change `new HumanTaskCandidates(groups, users)` to
`HumanTaskCandidates.of(groups, users)`.

Update `ConstraintHumanTaskRoutingStrategyTest.candidates()` helper:
Change `new HumanTaskCandidates(groups, users)` to
`HumanTaskCandidates.of(groups, users)`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=HumanTaskCandidatesTest`
Expected: all PASS

- [ ] **Step 6: Build full project to verify no compile errors**

Run: `ide_build_project`
Expected: success, no errors

- [ ] **Step 7: Commit**

```
feat(#757): add groupMembership and allUsers() to HumanTaskCandidates

Refs #757
```

---

### Task 2: ContextConstraint.Builder accumulation fix

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/routing/ContextConstraint.java`
- Create: `api/src/test/java/io/casehub/api/model/routing/ContextConstraintBuilderTest.java`

**Interfaces:**
- Produces: `Builder.prefer(Set<String> groups, Set<String> users)` → `Prefer(groups, users)`
- Produces: `Builder.exclude(Set<String> groups, Set<String> users)` → `Exclude(groups, users)`
- Modified: `preferGroups()` / `preferUsers()` accumulate within same effect type
- Modified: `excludeGroups()` / `excludeUsers()` accumulate within same effect type

- [ ] **Step 1: Write failing tests for builder accumulation**

Create test file `api/src/test/java/io/casehub/api/model/routing/ContextConstraintBuilderTest.java`:

```java
package io.casehub.api.model.routing;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Set;
import org.junit.jupiter.api.Test;

class ContextConstraintBuilderTest {

  @Test
  void preferGroupsThenPreferUsersAccumulates() {
    var c = ContextConstraint.builder()
        .when(".always.true")
        .preferGroups(Set.of("managers"))
        .preferUsers(Set.of("alice"))
        .build();
    assertThat(c.effect()).isInstanceOf(ContextConstraint.Prefer.class);
    var prefer = (ContextConstraint.Prefer) c.effect();
    assertThat(prefer.groups()).containsExactly("managers");
    assertThat(prefer.users()).containsExactly("alice");
  }

  @Test
  void preferGroupsRepeatedAccumulates() {
    var c = ContextConstraint.builder()
        .when(".always.true")
        .preferGroups(Set.of("managers"))
        .preferGroups(Set.of("leads"))
        .build();
    var prefer = (ContextConstraint.Prefer) c.effect();
    assertThat(prefer.groups()).containsExactlyInAnyOrder("managers", "leads");
  }

  @Test
  void excludeGroupsThenExcludeUsersAccumulates() {
    var c = ContextConstraint.builder()
        .when(".always.true")
        .excludeGroups(Set.of("interns"))
        .excludeUsers(Set.of("bob"))
        .build();
    var exclude = (ContextConstraint.Exclude) c.effect();
    assertThat(exclude.groups()).containsExactly("interns");
    assertThat(exclude.users()).containsExactly("bob");
  }

  @Test
  void switchingEffectTypeReplaces() {
    var c = ContextConstraint.builder()
        .when(".always.true")
        .preferGroups(Set.of("managers"))
        .excludeUsers(Set.of("bob"))
        .build();
    assertThat(c.effect()).isInstanceOf(ContextConstraint.Exclude.class);
    var exclude = (ContextConstraint.Exclude) c.effect();
    assertThat(exclude.users()).containsExactly("bob");
    assertThat(exclude.groups()).isEmpty();
  }

  @Test
  void combinedPreferFactory() {
    var c = ContextConstraint.builder()
        .when(".always.true")
        .prefer(Set.of("managers"), Set.of("alice"))
        .build();
    var prefer = (ContextConstraint.Prefer) c.effect();
    assertThat(prefer.groups()).containsExactly("managers");
    assertThat(prefer.users()).containsExactly("alice");
  }

  @Test
  void combinedExcludeFactory() {
    var c = ContextConstraint.builder()
        .when(".always.true")
        .exclude(Set.of("interns"), Set.of("bob"))
        .build();
    var exclude = (ContextConstraint.Exclude) c.effect();
    assertThat(exclude.groups()).containsExactly("interns");
    assertThat(exclude.users()).containsExactly("bob");
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=ContextConstraintBuilderTest`
Expected: FAIL — `preferGroupsThenPreferUsersAccumulates` fails because
`preferUsers()` overwrites the effect

- [ ] **Step 3: Implement builder accumulation**

Replace the Builder class in `ContextConstraint.java` using `ide_edit_member`
(member=`Builder`):

```java
public static final class Builder {
  private ExpressionEvaluator condition;
  private Effect effect;
  private double weight = 1.0;

  private Builder() {}

  public Builder when(String jqExpression) {
    this.condition = new JQExpressionEvaluator(jqExpression);
    return this;
  }

  public Builder when(Predicate<CaseContext> predicate) {
    this.condition = new LambdaExpressionEvaluator(predicate);
    return this;
  }

  public Builder when(ExpressionEvaluator evaluator) {
    this.condition = evaluator;
    return this;
  }

  public Builder preferUsers(Set<String> users) {
    if (effect instanceof Prefer existing) {
      this.effect = new Prefer(existing.groups(), union(existing.users(), users));
    } else {
      this.effect = new Prefer(Set.of(), users);
    }
    return this;
  }

  public Builder preferGroups(Set<String> groups) {
    if (effect instanceof Prefer existing) {
      this.effect = new Prefer(union(existing.groups(), groups), existing.users());
    } else {
      this.effect = new Prefer(groups, Set.of());
    }
    return this;
  }

  public Builder prefer(Set<String> groups, Set<String> users) {
    this.effect = new Prefer(groups, users);
    return this;
  }

  public Builder excludeUsers(Set<String> users) {
    if (effect instanceof Exclude existing) {
      this.effect = new Exclude(existing.groups(), union(existing.users(), users));
    } else {
      this.effect = new Exclude(Set.of(), users);
    }
    return this;
  }

  public Builder excludeGroups(Set<String> groups) {
    if (effect instanceof Exclude existing) {
      this.effect = new Exclude(union(existing.groups(), groups), existing.users());
    } else {
      this.effect = new Exclude(groups, Set.of());
    }
    return this;
  }

  public Builder exclude(Set<String> groups, Set<String> users) {
    this.effect = new Exclude(groups, users);
    return this;
  }

  public Builder weight(double weight) {
    this.weight = weight;
    return this;
  }

  public ContextConstraint build() {
    if (condition == null) {
      throw new IllegalStateException("condition is required");
    }
    if (effect == null) {
      throw new IllegalStateException(
          "effect is required — call preferUsers(), preferGroups(), excludeUsers(), or excludeGroups()");
    }
    return new ContextConstraint(condition, effect, weight);
  }

  private static Set<String> union(Set<String> a, Set<String> b) {
    var result = new java.util.LinkedHashSet<>(a);
    result.addAll(b);
    return Set.copyOf(result);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=ContextConstraintBuilderTest`
Expected: all PASS

- [ ] **Step 5: Commit**

```
feat(#757): fix ContextConstraint.Builder to accumulate group+user effects

Refs #757
```

---

### Task 3: CBR strategy scores allUsers()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrHumanTaskRoutingStrategy.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrHumanTaskRoutingStrategyTest.java`

**Interfaces:**
- Consumes: `HumanTaskCandidates.allUsers()`, `HumanTaskCandidates.of()` from Task 1

- [ ] **Step 1: Add test helper for candidates with group membership**

Add a second `candidates` helper to `CbrHumanTaskRoutingStrategyTest`:

```java
private HumanTaskCandidates candidates(
    Set<String> groups, Set<String> users, Map<String, Set<String>> groupMembership) {
  return new HumanTaskCandidates(groups, users, groupMembership);
}
```

- [ ] **Step 2: Write failing tests for group-expanded scoring**

```java
@Test
void scoresGroupExpandedUsers() {
  var exp = experience(0.9, step("review-task", "bob", "SUCCESS"));
  var result = strategy.select(
      context("review-task", List.of(exp)),
      candidates(Set.of("managers"), Set.of("alice"),
          Map.of("managers", Set.of("bob", "charlie"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateScores()).containsEntry("bob", 1.0);
  assertThat(enriched.candidateScores()).doesNotContainKey("charlie");
}

@Test
void emptyAllUsersReturnsUnchanged() {
  var exp = experience(0.9, step("review-task", "alice", "SUCCESS"));
  var result = strategy.select(
      context("review-task", List.of(exp)),
      candidates(Set.of("managers"), Set.of(), Map.of()));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Unchanged.class);
}

@Test
void enrichedCandidateUsersContainsAllUsers() {
  var exp = experience(0.9, step("review-task", "bob", "SUCCESS"));
  var result = strategy.select(
      context("review-task", List.of(exp)),
      candidates(Set.of("managers"), Set.of("alice"),
          Map.of("managers", Set.of("bob"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateUsers()).containsExactlyInAnyOrder("alice", "bob");
}

@Test
void overlapBetweenDirectAndGroupProducesSingleScore() {
  var exp = experience(0.9, step("review-task", "alice", "SUCCESS"));
  var result = strategy.select(
      context("review-task", List.of(exp)),
      candidates(Set.of("managers"), Set.of("alice"),
          Map.of("managers", Set.of("alice", "bob"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateScores()).containsEntry("alice", 1.0);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=CbrHumanTaskRoutingStrategyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `scoresGroupExpandedUsers` fails (bob not scored because he's not in `candidates.users()`)

- [ ] **Step 4: Update CbrHumanTaskRoutingStrategy.select()**

Use `ide_replace_member` on `select` method:

```java
@Override
public HumanTaskRoutingResult select(
    final HumanTaskRoutingContext context, final HumanTaskCandidates candidates) {
  if (context.experiences().isEmpty()) {
    return new HumanTaskRoutingResult.Unchanged();
  }

  final Set<String> allUsers = candidates.allUsers();
  if (allUsers.isEmpty()) {
    return new HumanTaskRoutingResult.Unchanged();
  }

  final String bindingName = context.bindingName();
  final Map<String, Double> scores =
      ExperienceAnalyser.workerSuccessRates(
          context.experiences(),
          allUsers,
          step -> bindingName.equals(step.bindingName()),
          ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);

  if (scores.isEmpty()) {
    return new HumanTaskRoutingResult.Unchanged();
  }

  return new HumanTaskRoutingResult.Enriched(candidates.groups(), allUsers, scores);
}
```

- [ ] **Step 5: Update existing test `emptyUsersReturnsUnchanged`**

The test passes `candidates(Set.of("managers"), Set.of())` — with no
group membership, `allUsers()` returns the empty direct users. Still valid.

Update the test name and the deferred-no-op test `groupEffectsDeferredNoOp`
in `ConstraintHumanTaskRoutingStrategyTest` — this will be replaced in Task 5.

- [ ] **Step 6: Run all tests to verify**

Run: `mvn test -pl runtime -Dtest=CbrHumanTaskRoutingStrategyTest`
Expected: all PASS

- [ ] **Step 7: Commit**

```
feat(#757): CBR humanTask strategy scores group-expanded users

Refs #757
```

---

### Task 4: Constraint strategy applies group effects

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/ConstraintHumanTaskRoutingStrategy.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/ConstraintHumanTaskRoutingStrategyTest.java`

**Interfaces:**
- Consumes: `HumanTaskCandidates.allUsers()`, `HumanTaskCandidates.groupMembership()` from Task 1
- Consumes: `ContextConstraint.Builder.prefer()`, `.exclude()`, accumulation from Task 2

- [ ] **Step 1: Add test helper for candidates with group membership**

Add a second `candidates` helper to `ConstraintHumanTaskRoutingStrategyTest`:

```java
private HumanTaskCandidates candidates(
    Set<String> groups, Set<String> users, Map<String, Set<String>> groupMembership) {
  return new HumanTaskCandidates(groups, users, groupMembership);
}
```

- [ ] **Step 2: Write failing tests for group effects**

```java
@Test
void excludeGroupRemovesGroupAndMembers() {
  expressionRegistry.nextResult = true;
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskContextConstraint(
          ContextConstraint.builder()
              .when(".always.true")
              .excludeGroups(Set.of("interns"))
              .weight(1.0)
              .build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("interns", "managers"), Set.of("alice"),
          Map.of("interns", Set.of("bob", "charlie"),
                 "managers", Set.of("alice"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateGroups()).containsExactly("managers");
  assertThat(enriched.candidateUsers()).doesNotContain("bob", "charlie");
  assertThat(enriched.candidateUsers()).contains("alice");
}

@Test
void excludeGroupOverridesDirectNomination() {
  expressionRegistry.nextResult = true;
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskContextConstraint(
          ContextConstraint.builder()
              .when(".always.true")
              .excludeGroups(Set.of("blocked"))
              .weight(1.0)
              .build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("blocked"), Set.of("alice"),
          Map.of("blocked", Set.of("alice"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Escalated.class);
}

@Test
void preferGroupBoostsMemberScores() {
  expressionRegistry.nextResult = true;
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskContextConstraint(
          ContextConstraint.builder()
              .when(".always.true")
              .preferGroups(Set.of("seniors"))
              .weight(0.7)
              .build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("seniors"), Set.of("alice"),
          Map.of("seniors", Set.of("bob", "charlie"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateScores()).containsEntry("bob", 0.7);
  assertThat(enriched.candidateScores()).containsEntry("charlie", 0.7);
  assertThat(enriched.candidateScores()).doesNotContainKey("alice");
}

@Test
void preferWithUserInBothGroupAndUsersAppliesWeightOnce() {
  expressionRegistry.nextResult = true;
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskContextConstraint(
          ContextConstraint.builder()
              .when(".always.true")
              .prefer(Set.of("seniors"), Set.of("alice"))
              .weight(0.5)
              .build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("seniors"), Set.of("alice"),
          Map.of("seniors", Set.of("alice", "bob"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateScores().get("alice")).isCloseTo(0.5, within(0.001));
}

@Test
void allExcludedViaGroupsEscalates() {
  expressionRegistry.nextResult = true;
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskContextConstraint(
          ContextConstraint.builder()
              .when(".always.true")
              .excludeGroups(Set.of("everyone"))
              .weight(1.0)
              .build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("everyone"), Set.of(),
          Map.of("everyone", Set.of("alice", "bob"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Escalated.class);
}

@Test
void workloadAppliesToGroupExpandedUsers() {
  workloadProvider.workload = Map.of(
      "alice", new WorkloadSnapshot(2),
      "bob", new WorkloadSnapshot(8));
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskWorkloadConstraint(WorkloadConstraint.builder().maxActiveTaskCount(5).build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("team"), Set.of(),
          Map.of("team", Set.of("alice", "bob"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateUsers()).contains("alice");
  assertThat(enriched.candidateUsers()).doesNotContain("bob");
}

@Test
void eligibleUsersInitializedFromAllUsers() {
  expressionRegistry.nextResult = true;
  var def = CaseDefinition.builder()
      .namespace("test").name("test").version("1.0")
      .humanTaskContextConstraint(
          ContextConstraint.builder()
              .when(".always.true")
              .preferUsers(Set.of("bob"))
              .weight(0.6)
              .build())
      .build();
  var result = strategy.select(context(def),
      candidates(Set.of("team"), Set.of("alice"),
          Map.of("team", Set.of("bob"))));
  assertThat(result).isInstanceOf(HumanTaskRoutingResult.Enriched.class);
  var enriched = (HumanTaskRoutingResult.Enriched) result;
  assertThat(enriched.candidateScores()).containsEntry("bob", 0.6);
  assertThat(enriched.candidateUsers()).containsExactlyInAnyOrder("alice", "bob");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=ConstraintHumanTaskRoutingStrategyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — group effects not implemented

- [ ] **Step 4: Update ConstraintHumanTaskRoutingStrategy.select()**

Use `ide_replace_member` on `select`:

```java
@Override
public HumanTaskRoutingResult select(
    final HumanTaskRoutingContext context, final HumanTaskCandidates candidates) {
  final var definition = context.caseDefinition();
  if (definition == null) {
    return new HumanTaskRoutingResult.Unchanged();
  }

  final var contextConstraints = definition.getHumanTaskContextConstraints();
  final var workloadConstraint = definition.getHumanTaskWorkloadConstraint();

  if (contextConstraints.isEmpty() && workloadConstraint == null) {
    return new HumanTaskRoutingResult.Unchanged();
  }

  final Set<String> eligibleUsers = new LinkedHashSet<>(candidates.allUsers());
  final Set<String> eligibleGroups = new LinkedHashSet<>(candidates.groups());
  final Map<String, Double> scores = new HashMap<>();

  applyContextConstraints(
      contextConstraints, context, candidates, eligibleUsers, eligibleGroups, scores);

  if (eligibleUsers.isEmpty()) {
    return new HumanTaskRoutingResult.Escalated("all candidates excluded by context constraints");
  }

  if (workloadConstraint != null) {
    applyWorkloadConstraint(workloadConstraint, eligibleUsers, scores, context.tenancyId());

    if (eligibleUsers.isEmpty()) {
      return new HumanTaskRoutingResult.Escalated(
          "all candidates excluded by workload constraints");
    }
  }

  scores.keySet().retainAll(eligibleUsers);

  final boolean usersChanged = !eligibleUsers.equals(candidates.allUsers());
  final boolean groupsChanged = !eligibleGroups.equals(candidates.groups());
  if (!usersChanged && !groupsChanged && scores.isEmpty()) {
    return new HumanTaskRoutingResult.Unchanged();
  }

  return new HumanTaskRoutingResult.Enriched(eligibleGroups, eligibleUsers, scores);
}
```

- [ ] **Step 5: Update applyContextConstraints to handle group effects**

Use `ide_replace_member` on `applyContextConstraints`:

```java
private void applyContextConstraints(
    final java.util.List<ContextConstraint> constraints,
    final HumanTaskRoutingContext context,
    final HumanTaskCandidates candidates,
    final Set<String> eligibleUsers,
    final Set<String> eligibleGroups,
    final Map<String, Double> scores) {
  for (final ContextConstraint constraint : constraints) {
    boolean match;
    try {
      match = expressionRegistry.evaluate(constraint.condition(), context.caseContext());
    } catch (final Exception e) {
      LOG.log(
          System.Logger.Level.WARNING,
          "Constraint condition evaluation failed — treating as false",
          e);
      match = false;
    }

    if (!match) {
      continue;
    }

    switch (constraint.effect()) {
      case ContextConstraint.Exclude exclude -> {
        eligibleUsers.removeAll(exclude.users());
        for (final String group : exclude.groups()) {
          eligibleGroups.remove(group);
          final Set<String> members = candidates.groupMembership().getOrDefault(group, Set.of());
          eligibleUsers.removeAll(members);
        }
      }
      case ContextConstraint.Prefer prefer -> {
        final Set<String> usersToBoost = new LinkedHashSet<>(prefer.users());
        for (final String group : prefer.groups()) {
          usersToBoost.addAll(
              candidates.groupMembership().getOrDefault(group, Set.of()));
        }
        for (final String user : usersToBoost) {
          if (eligibleUsers.contains(user)) {
            scores.merge(user, constraint.weight(), Double::sum);
          }
        }
      }
    }
  }
}
```

- [ ] **Step 6: Remove the deferred-no-op test**

Delete test `groupEffectsDeferredNoOp` from `ConstraintHumanTaskRoutingStrategyTest`
— group effects are now functional, this test asserts the old deferred behavior.

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl runtime -Dtest=ConstraintHumanTaskRoutingStrategyTest`
Expected: all PASS

- [ ] **Step 8: Build to verify no compile errors**

Run: `ide_build_project`
Expected: success

- [ ] **Step 9: Commit**

```
feat(#757): constraint strategy applies group Exclude/Prefer effects

Refs #757
```

---

### Task 5: Handler expands groups via GroupMembershipProvider

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

**Interfaces:**
- Consumes: `GroupMembershipProvider.membersOf(String, String)` from `casehub-platform-api`
- Consumes: `HumanTaskCandidates(Set, Set, Map)` three-arg constructor from Task 1

- [ ] **Step 1: Add GroupMembershipProvider injection**

Use `ide_insert_member` after the `evaluationSerializer` field:

```java
@Inject io.casehub.platform.api.identity.GroupMembershipProvider groupMembershipProvider;
```

- [ ] **Step 2: Update publishHumanTaskSchedule to expand groups**

Replace the section where `htCandidates` is constructed (after `resolvedUsers`
is computed, before `humanTaskStrategy.select()`). Use `ide_replace_text_in_file`
to change:

```java
final var htCandidates = new HumanTaskCandidates(resolvedGroups, resolvedUsers);
```

to:

```java
final Map<String, Set<String>> groupMembership = expandGroupMembership(
    resolvedGroups, caseInstance.tenancyId);
final var htCandidates = new HumanTaskCandidates(
    resolvedGroups, resolvedUsers, groupMembership);
```

- [ ] **Step 3: Add expandGroupMembership method**

Use `ide_insert_member` after `publishHumanTaskSchedule`:

```java
private Map<String, Set<String>> expandGroupMembership(
    final Set<String> groups, final String tenancyId) {
  if (groups == null || groups.isEmpty()) {
    return Map.of();
  }
  final var membership = new java.util.LinkedHashMap<String, Set<String>>();
  for (final String group : groups) {
    try {
      final var members = groupMembershipProvider.membersOf(group, tenancyId);
      if (members != null && !members.isEmpty()) {
        membership.put(
            group,
            members.stream()
                .map(io.casehub.platform.api.identity.GroupMember::actorId)
                .collect(java.util.stream.Collectors.toUnmodifiableSet()));
      }
    } catch (final Exception e) {
      LOG.warnf(
          e,
          "Group expansion failed for group '%s' tenancyId=%s — treating as empty",
          group,
          tenancyId);
    }
  }
  return Map.copyOf(membership);
}
```

- [ ] **Step 4: Build to verify**

Run: `ide_build_project`
Expected: success

- [ ] **Step 5: Commit**

```
feat(#757): handler expands candidate groups via GroupMembershipProvider

Refs #757
```

---

### Task 6: Update HumanTaskRoutingResult javadoc and CLAUDE.md

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/HumanTaskRoutingResult.java`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update HumanTaskRoutingResult javadoc**

Use `ide_edit_member` (member=`HumanTaskRoutingResult`) to update the class-level
javadoc, removing the engine#757 caveat and documenting the invariant:

```java
/**
 * Sealed result type from {@link HumanTaskRoutingStrategy#select}. Follows the convention of {@link
 * RoutingResult} (Selected | Unresolvable | Escalated) and {@link ImplementationSelection}
 * (Selected | RunAll | RunNone).
 *
 * <p>{@code candidateScores} keys are individual actor IDs (direct or group-expanded), never group
 * names. Invariant: {@code candidateScores.keySet() ⊆ candidateUsers}.
 */
```

- [ ] **Step 2: Update CLAUDE.md HumanTaskRoutingStrategy SPI section**

Update the section to reflect group scoring is now implemented. Key changes:
- `HumanTaskCandidates` carries `groupMembership` and `allUsers()`
- `candidateScores` keys include group-expanded users
- Both strategies use `allUsers()` for scoring
- Constraint strategy applies group Exclude/Prefer effects
- Handler expands groups via `GroupMembershipProvider`

- [ ] **Step 3: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime`
Expected: all PASS

- [ ] **Step 4: Commit**

```
feat(#757): update javadoc and CLAUDE.md for group scoring

Closes #757
```
