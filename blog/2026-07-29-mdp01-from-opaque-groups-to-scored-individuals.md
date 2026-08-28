---
layout: post
title: "From Opaque Groups to Scored Individuals"
date: 2026-07-29
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [routing, humantask, cbr, group-scoring]
series: issue-797-humantask-cbr-routing
---

*Part of a series on [#797 — HumanTask CBR routing](https://github.com/casehubio/engine/issues/797). Previous: [Teaching the Engine Who Should Handle This](2026-07-29-teaching-the-engine-who-should-handle-this.md).*

When a workflow engine assigns a task to a human, it typically resolves who should see it from a candidate set — a list of users, a list of groups, or both. The candidate set answers "who *can* do this." It doesn't answer "who *should* do this."

That second question is where routing strategies come in. The casehub engine supports pluggable `HumanTaskRoutingStrategy` implementations that enrich the candidate set with scores before the task reaches the work inbox. A CBR strategy retrieves similar past cases and scores candidates by historical success rate. A constraint strategy applies declarative rules — boost users matching a context condition, exclude users who are overloaded, escalate when nobody qualifies.

Both strategies had the same blind spot: they could only score individually-nominated users. Groups were opaque labels. If you assigned a task to "legal-team," the engine passed that label through to the work repo unchanged. The work repo knows who's in the group — it resolves membership at assignment time — but the engine's scoring layer never saw the individuals. A team of five with very different track records all looked the same: unscored.

This matters because group-based assignment is the norm, not the exception. Most organisations don't nominate individuals for every task — they nominate roles, teams, departments. If your routing strategy can't see through groups to the people inside them, it's scoring a minority of candidates and leaving the majority to luck.

## The platform already had the answer

I expected to need a new SPI for group membership resolution. Instead, `GroupMembershipProvider` was already sitting in `casehub-platform-api` — `membersOf(groupName, tenancyId) → Set<GroupMember>`. The work repo's `TemplateExpander` uses it to resolve `excludedGroups` into actor IDs for WorkItem templates. Same problem, same SPI, different call site.

The design question was where to call it. Three options: expand in the handler (single point, before strategy dispatch), inject into each strategy (lazy, but duplicated), or pass membership on the routing context (semantically wrong — membership is about the candidates, not the case).

We went with handler expansion. `CaseContextChangedEventHandler.publishHumanTaskSchedule()` now calls `membersOf()` for each candidate group and passes the resolved membership as a `Map<String, Set<String>>` on `HumanTaskCandidates`. A new `allUsers()` method returns the union of direct users and group members — strategies score the full set without caring how each user was nominated. Per-group error isolation means a failed directory lookup for one group doesn't block the others or the directly-nominated users.

## The invariant that fell out of the design review

The adversarial review caught an inconsistency I'd missed. The original spec said `Enriched.candidateUsers()` should be "original direct users only" — but the constraint strategy was initialising its eligible set from `allUsers()` (direct + group-expanded). If it then returned `eligibleUsers` in the result, the scores map could contain keys not present in `candidateUsers`. The invariant `candidateScores.keySet() ⊆ candidateUsers` would break.

The fix was clean: both strategies now return `allUsers()` as `candidateUsers` in the `Enriched` result. Scores are always a subset. The work repo receives the full individually-eligible set and matches scores by actor ID after its own group resolution.

## A pre-existing builder bug

The design review also surfaced that `ContextConstraint.Builder` had an overwrite bug. Calling `preferGroups(g).preferUsers(u)` silently discarded the groups — each method replaced the entire `effect` field rather than accumulating within the same effect type. This was harmless when group effects were no-ops, but became a live bug the moment they became functional. The fix: same-type calls accumulate (`preferGroups(g1).preferGroups(g2)` → `Prefer(g1 ∪ g2, ∅)`), cross-type calls replace (consistent with the sealed effect semantics).

The constraint strategy now removes excluded groups from `candidateGroups` in the result — without this, the work repo would override the engine's exclusion by showing the task to group members anyway. Group exclusion is a policy override: it removes members regardless of whether they were also directly nominated.

There's a deliberate TOCTOU gap between the engine's group resolution (at scoring time) and the work repo's group resolution (at assignment time). Membership can change between the two. We accepted this because scores are advisory signals for ranking, not gates for eligibility. A user added to a group after scoring has no score but remains assignable. A user removed retains a stale score that's harmless — the work repo is authoritative for who can actually claim the task.
