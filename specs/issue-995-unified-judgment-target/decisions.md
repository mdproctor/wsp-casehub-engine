## D1: Unified JudgmentTarget with RoutingConfig — delete HumanTaskTarget

**Choice:** JudgmentTarget becomes THE unified yield target. HumanTaskTarget is deleted. Caller-type-specific routing hints move to a `RoutingConfig` sealed interface (nullable) on JudgmentTarget. `HumanRoutingConfig` carries templateRef, candidateGroups, candidateUsers, claimDeadlineHours, payloadType. Yield-semantic fields (title, outcomes, scope, priority) move from HumanTaskTarget to JudgmentTarget directly. `DelegatingJudgmentScheduler` replaces `NoOpJudgmentScheduler` and delegates to `HumanTaskScheduler` when HumanRoutingConfig is present.
**Alternatives:**
- Shared `YieldTarget` interface keeping both types — doesn't unify dispatch/completion/verification paths; two parallel paths persist
- Flatten all fields onto JudgmentTarget as nullable — 20+ fields with no type-level guidance; no extensibility for future caller types
- Keep separate types (status quo from #996 D1) — pragmatic rationale that doesn't hold under "fix the design" directive
**Rationale:** First-principles separation: WHAT (yield semantics on JudgmentTarget) vs WHO (routing hints on RoutingConfig) vs HOW VERIFIED (verifier on JudgmentTarget). Sealed RoutingConfig extensible for future caller types without touching target. Verification, evidence, and provenance apply uniformly to ALL yields. One dispatch path, one completion path, one escalation path.
**Trade-offs:** Significant refactoring — delete HumanTaskTarget, update 6 switch sites, rewrite publishHumanTaskSchedule, update YAML schema. All existing `humanTask:` YAML definitions need updating to `judgment:` with `human:` sub-block. Pre-release: cost is irrelevant.
**Sources:** `HumanTaskTarget.java` (16+ fields mixing concerns), `CaseContextChangedEventHandler.java:685-815` (publishHumanTaskSchedule), `BindingTarget.java:27` (sealed permits), Epic #994 (caller-agnostic vision), first-principles yield analysis
**Exploration:** deep-analysis
**Status:** captured
