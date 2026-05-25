---
layout: post
title: "The List That Emptied Itself"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [workflow, handover, multi-session]
---

The "What's Next" table in my handover had nine items. I went to start the first one — engine#300, add the deadline field to the COMMAND message — and found the issue closed. The code was already there, committed three sessions back. All nine items were gone.

This is structural. The engine session tracks its own state, but much of the work it depends on — protocols, platform docs, cross-repo standards — happens in the parent session, which has its own handover. When something closes there, the engine handover doesn't know. The list goes stale silently.

There's no elegant fix. The handover is a snapshot, not a live view.

The other loose end was a workspace branch — `issue-6-sla-propagation` — left open for several sessions. It had zero project commits. The SLA propagation code landed directly on engine main in an engine session, bypassing the parent tracking branch entirely. The parent session opened the branch to manage the cross-repo concern but never closed it, because there was nothing to merge.

Running `work-end` for a branch owned by another workspace is subtler than it looks. The skill resolves paths via `readlink -f proj` and `git rev-parse --show-toplevel` — both relative to the current session's working directory. Invoke it naively from the engine session and it operates on the engine repos while believing it's working on the parent. No error, just the wrong thing happening quietly.
