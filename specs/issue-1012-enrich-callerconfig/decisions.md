# Decisions — engine#1012 Enrich Judgment Foundation Types

## D1: Evidence field naming and typing

**Choice:** Full v2 alignment — rename `key`→`name`, `value:Object`→`content:String`, add `ref:@Nullable String`
**Alternatives:**
- Add ref only — keeps field names stable but leaves Object typing as technical debt
- Narrow type only — fixes typing but keeps inconsistent naming vs v2
**Rationale:** Pre-release with no external consumers. Clean break costs nothing now, costs migration later.
**Trade-offs:** All existing Evidence construction sites need updating (field renames).
**Sources:** v2 branch `6464141f`, issue #1012 body
**Exploration:** quick
**Status:** captured

## D2: CallerIdentity nullability

**Choice:** Required `callerId`/`callerType` (v2 design), add `@Nullable Double trustScore`, remove `anonymous()`
**Alternatives:**
- Keep nullable + add trustScore — preserves anonymous() but muddies the identity contract
**Rationale:** If there's no caller, `callerIdentity` is null at the call site. A CallerIdentity with null callerId is a meaningless object. Required fields enforce that if you have an identity, it's complete.
**Trade-offs:** VerificationContext/EscalationContext backward-compat constructors that synthesised CallerIdentity from nullable strings must be removed.
**Sources:** v2 branch `6464141f`, current VerificationContext.java
**Exploration:** quick
**Status:** captured

## D3: No backward compatibility — clean architecture throughout

**Choice:** Remove all backward-compat constructors, remove raw `callerId`/`callerType` string fields from VerificationContext/EscalationContext, remove untyped `Map<String, Object> evidence` in favour of `List<Evidence>` only. Refactor all consumers.
**Alternatives:**
- Preserve backward compat — keeps old constructors, defers cleanup
**Rationale:** Pre-release, no external consumers. Backward-compat constructors are dead weight that obscure the real API.
**Trade-offs:** More files changed in this PR. All construction sites across engine, runtime, work-cloudevent, and tests need updating.
**Sources:** User directive — "design for best architecture, no technical debt, refactor across repos"
**Exploration:** quick
**Status:** captured

## D4: CallerConfig.Llm field naming

**Choice:** Keep main's `modelId` (not v2's `model`) alongside new `modelName` and `systemPrompt`
**Alternatives:**
- Use v2's `model` — shorter but ambiguous with the `ChatModel` type
**Rationale:** `modelId` is self-documenting and avoids type confusion.
**Trade-offs:** None — v2's `model` field had no consumers.
**Sources:** Current CallerConfig.java, v2 branch `6464141f`
**Exploration:** quick
**Status:** captured
