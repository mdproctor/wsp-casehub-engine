# Handoff — CDI Test Fixes + Pin Move
2026-05-21

## What changed this session

**1 issue closed: #277.**

Moved json-schema-validator 1.5.4 pin from `work-adapter/pom.xml` to root `pom.xml` `<dependencyManagement>`. Verification uncovered three pre-existing test failures fixed in the same commit:
- `getEnabledAlternatives()` replacing (not appending) `selected-alternatives` → atomicity test had 32 CDI deployment failures
- `@Alternative @Priority(1)` from external JARs not overriding non-alternative beans in Quarkus ARC 3.x → `JpaWorkItemStore` was silently winning over `InMemoryWorkItemStore`; fixed with `quarkus.arc.exclude-types`
- CDI proxy instanceof check failing → store clear was a no-op

CLAUDE.md updated with the JpaWorkItemStore exclude-types and `getEnabledAlternatives()` replacement semantics. Two garden entries filed (GE-20260521-3ce7ca, GE-20260521-4de4f1).

**PR #313 open:** https://github.com/casehubio/engine/pull/313 — 30 commits ahead of upstream/main.

## Immediate Next Step

Review and merge PR #313: `gh pr view 313 --repo casehubio/engine`

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| #253 | Assess quarkus-hibernate-reactive-panache compile-scope dep | S | Med | — |
| work#174 | DB-level UNIQUE on WorkItemTemplate.name | S | Low | — |
| work#175 | JSON merge semantics defaultPayload + inputData | S | Med | — |
| Devtown | Epic 3: CasePlanModel PR review | M | Med | Queued multiple sessions |

## Key references

- Blog: `blog/2026-05-21-mdp01-pin-was-two-lines-cdi-was-not.md`
- Garden: GE-20260521-3ce7ca (`@Alternative @Priority` external jar), GE-20260521-4de4f1 (`getEnabledAlternatives()` replaces)
- PR: https://github.com/casehubio/engine/pull/313
