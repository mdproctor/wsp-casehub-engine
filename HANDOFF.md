# HANDOFF — casehub-engine

## Last Session

Closed all 4 follow-up issues from #726 code review: #1325, #1326, #1327, #1328. All work committed directly to main (XS follow-ups, no feature branch). Published to origin.

### What Landed

5 commits on engine main (SHA range `784ee30ed`..`80fca5a07`):

1. **#1325 — REINVOKED test coverage:** 3 tests for the previously untested REINVOKED execution path in `SyncWorkerFunctionHandler` — accumulated state round-trip, binding lock release on success, lock release on failure.
2. **#1326 — Timeout enrichment integration test:** Verifies `WorkerExecutionConfig.withEffectiveTimeout()` enrichment propagates through `wrapFunction` to the platform executor. Captures the Worker at the platform executor and asserts its ExecutionPolicy has the enriched default timeout.
3. **#1327 — Drop redundant `outputProjection` param:** Removed `String outputProjection` from `WorkerExecutor.execute()`. `DefaultWorkerExecutor` now reads from `capability.outputProjection()` directly. Interface simplified from 6 args to 5. 4 files updated.
4. **#1328 — Agent handler Guard/OTel decision:** Closed with decision: keep as-is. Agent invocations have richer observability (token counts, cost, latency), their own timeout, and retry would be harmful (LLM calls cost money). Rationale documented in the issue comment.
5. **Hook fix:** Removed `--no-verify` bypass advice from pre-push hooks (engine + workspace) — it triggered `canonical_sync.py`'s retired-hook detection, blocking all publishes.

### Blog

"After the Split: What the Edges Revealed" — covers what code review surfaced from the handler split: untested REINVOKED mode, redundant parameter, and the deliberate boundary decision for agent handlers. SVG diagram. At `docs/blog/2026-11-19-mdp01-after-the-split-what-the-edges-revealed.md`.

### Code Review Findings Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| 1329 | Remove dead 30_000 timeout fallback from AgentWorkerFunctionHandler | XS | Low |
| 1330 | Add outputProjection integration test through Capability path | XS | Low |

Finding #1329: The orchestrator always enriches `ExecutionPolicy` before dispatch, making the agent handler's `30_000`ms hardcoded fallback unreachable.

Finding #1330: After #1327, `Capability.outputProjection()` is never null (`Objects.requireNonNull` in compact constructor). The timeout test passes because `applyOutputSchema` short-circuits on null *output*, not on the projection string — but no test verifies the non-null JQ evaluation path through the new code.

## Queue Status

Queue drained. No remaining items. #1329 and #1330 are open on GitHub for future work.

## Published

Engine: 5 commits pushed to origin (SHA `80fca5a07`).
Workspace: 1 commit pushed to origin (SHA `ffd06a63`).

## Pre-Existing State

- Engine has unstaged changes from prior sessions across api, rest, persistence modules. These are NOT from this session — they predate #726 work. Some may be stashed changes that were popped; `git stash list` may still show older stashes.
- Workspace is on branch `feat/306-lipsync-warmup-delegation` (not main). Has 1 commit on that branch (hook fix). The workspace main was published but the feature branch was not.

## References

- `docs/blog/2026-11-19-mdp01-after-the-split-what-the-edges-revealed.md`
- `docs/blog/2026-11-18-mdp01-the-handler-that-stopped-reimplementing.md` (prior session)
- `docs/specs/issue-726-sync-handler-delegates-worker-executor/` (prior session)
