---
layout: post
title: "Cleaning House Before the Merge"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [git, pull-request, compaction]
---

The squash policy has existed as a document since early May. This session it
got its first application to live PRs.

PR #232 and #233 each had the same two-commit structure: a `feat:` commit with
an issue reference, followed by a `docs:` update to DESIGN.md and CLAUDE.md.
The policy is clear — docs follow-ons with no independent architectural decision
squash into the preceding feat commit.

```bash
git reset --soft HEAD~1
git commit --amend --no-edit
```

Two commands, no editor, message preserved. Both PRs collapsed to a single
commit each.

## The Hitchhiker

The compaction revealed something else. Claude ran
`git log upstream/main..feat/engine-221-message-type` after the squash and
got two commits back instead of one:

```
fa79673 feat: add MessageType to CaseChannelProvider.postToChannel() — engine#221
40bbc76 feat(api): add caseId to WorkResult for precise per-case listener lookup
```

The second commit didn't belong to this PR. It had been sitting on local
`main` — a WorkResult change not yet pushed upstream — and both feature branches
had been cut from local main rather than `upstream/main`. The commit rode along
silently into both PRs. The GitHub diff view didn't surface it clearly.

The fix: extract it to its own PR (#234), then rebase both branches onto
`upstream/main` to drop it.

```bash
git rebase --onto upstream/main 40bbc76 feat/engine-221-message-type
git rebase --onto upstream/main 40bbc76 feat/engine-229-provision-context-trigger
```

Three clean PRs — #232, #233, #234 — all rooted on upstream/main, one commit
each.

The rule is now in CLAUDE.md: cut feature branches from `upstream/main`, verify
with `git log upstream/main..branch` before pushing.
