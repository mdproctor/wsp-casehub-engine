---
layout: post
title: "Going Live — and the Two-Backup Mystery"
date: 2026-05-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [git, reconstruction, squash, history]
excerpt: "Pushing the reconstructed engine history live surfaces a silent rebase failure and a detective story about two backup branches, a stale fork, and feature code that never made it to casehubio."
---

The reconstruction was done. `main_proposal` had been sitting on `mdproctor/engine`
for a week — 115 commits, tree identical to `casehubio/engine`. I wanted a systematic
review before touching the canonical repo.

I ran the review with Claude against twelve specific checks: commit count, tree
integrity, absorbed SHAs, improved messages, group-specific correctness. Eleven passed.
One didn't.

## The Reword That Silently Skipped

Groups 1–3 in the compaction were supposed to reword three old GitHub merge headers —
"Merge pull request #36 from treblereel/issue29" and two like it — into something
meaningful. The commands were in the todo file. They just didn't fire.

In a non-interactive session, `reword` silently becomes `pick`. No editor opens, no
error, the original message stays. The rebase completes and there's no trace until
you grep for the expected new subjects and get nothing back.

All three were treblereel's PRs. His own GitHub PR titles were already precise, so
we reworded using those as the new subjects — `fix: prevent duplicate worker execution
from repeated signals (Closes #29)`, `refactor(api): rename DispatchRule → Binding...`,
`feat: add LambdaExpressionEvaluator and ExpressionEngine registry`. The
`exec git commit --amend -F /tmp/msg.txt` pattern (message in a temp file, bypassing
shell quoting entirely) handled the `→` character without trouble.

Engine went live on casehubio.

## The Stale Fork

Then the other five repos.

The plan: backup the current casehubio main, force-push the compacted branch. Claudony
had a zero-byte diff — clean, pushed immediately. Ledger stopped us cold.

`git diff upstream/main HEAD` came back with over a megabyte of content divergence.
That's not a history rewrite difference. The trees were genuinely different. Claude
flagged it and stopped: "Do not push ledger."

The first assumption was that casehubio had moved ahead while the squash sat idle.
We checked: 279 commits on casehubio not in the local backup, same pattern for work
(346), qhorus (337), parent (79). Hundreds of commits of drift.

Then we checked the *other* backup branch. Each repo had two: an original
(`backup/pre-squash-main-20260507`) and a v1 (`backup/pre-squash-v1-main-20260508`),
with no ancestor relationship between them — two separate squash attempts on two
different histories. When we checked the original against casehubio in both directions,
the picture flipped: casehubio had zero commits the original backup didn't have. The
original was *ahead* of casehubio, not behind.

The second squash session had based itself on `origin/main` — the mdproctor fork —
instead of `upstream/main`. The fork hadn't been synced; work had gone directly to
casehubio. The fork was 279–346 commits behind.

So the compacted branches weren't missing casehubio content. They had casehubio's
full history *plus* local feature code that had never been pushed —
`AttestationAggregator`, `LedgerComplianceReportService`, `ProvenanceCaptureInterceptor`
in ledger; the `register_backend` MCP tool and channel gateway in qhorus. Meanwhile
casehubio had accumulated methodology files — `HANDOFF.md`, blog entries — committed
directly to the project repo instead of the workspace where they belong.

Pushing the compacted branches would correct both: the real code goes in, the
methodology noise comes out. We created permanent backup branches on casehubio for
every repo — three per repo in some cases — then pushed all five. Zero-byte tree diffs
confirmed after each.

A day later the full-stack aggregator build exposed three compilation failures in
engine. A missing SPI method across four test stubs, and casehub-engine-common lacking
the serverlessworkflow dependencies needed to compile `WorkflowExecutor`. There was an
intermediate commit that moved the class before the proper fix added the missing deps —
we squashed that away and pushed the two clean commits forward to casehubio without a
force.
