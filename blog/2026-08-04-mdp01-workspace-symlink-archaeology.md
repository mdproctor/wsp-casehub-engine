---
layout: post
title: "Workspace Symlink Archaeology"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [workspace, diagnostics, tooling]
---

The engine's HANDOFF had work-repo issues in it. #329, #330, #298, #152, #328 — all from `casehubio/work`, not `casehubio/engine`. Branch detection was showing a work-repo branch as the active workspace state. Nothing was erroring. Everything looked plausible until you checked the issue numbers.

I asked Claude to verify the issues against the engine repo. Every one came back as a different issue entirely — engine #329 is "fix ledger ActorType import", not "Epic: Progress model enhancements". The HANDOFF had been written by a cross-repo triage session that was working on work-repo epics but writing to what it thought was the engine workspace.

The `wksp` symlink was pointing at `../work` — the work project's workspace root — instead of `../work/engine`, the engine's subdirectory within it. We traced the change through git:

```bash
git log --format="%h %ai %s" -- wksp
git show <commit>:wksp
```

The symlink had moved through four targets: an absolute path to a dedicated engine workspace (May), then `../work/engine` (July 21, correct), briefly `../wksp-work/engine` (July 29), and finally `../work` (August 2 — broken). The change was buried in a commit alongside substantive code changes. No error surfaced because the workspace directory existed at both paths — it just contained the wrong project's artifacts.

The engine's real workspace was sitting there at `/Users/mdproctor/claude/casehub/work/engine/` the whole time, with its own HANDOFF, blog entries, plans, and specs. The fix was `ln -sfn ../work/engine engine/wksp`.

The deeper fix was in the handover skill. The resume path already validates `#N` references against `OWNER_REPO` — it checks each issue exists in the project repo before presenting the handover. But the write path had no equivalent check. A triage session recalling issues from conversation context could write any repo's issues into any project's HANDOFF without validation. I added Step 5b to the write path: before committing, every `#N` in What's Left and What's Next gets verified against `OWNER_REPO` via `gh issue view`. Wrong-repo issues are silently removed.

While cleaning up, I also ran through the open issue backlog. Epic #797 (HumanTask CBR routing) had all four children closed — #754, #755, #757, and #756 — but the epic body still showed #756 unchecked. Closed it. Epic #800 (Agent Learning & Memory) is further along than the body suggests: both neocortex dependencies (#185 relationship memory, #186 reflective diary) have shipped, along with engine #784, #785, #807, and #860. Eight of the eighteen items are closed. The remaining eight engine issues — reflection orchestration, hierarchical planning, plan adaptation, goal formation and the rest — are now unblocked.

The symlink problem is a good reminder that silent failures in workspace tooling are the hardest kind to catch. The symptom pointed at the handover skill, at slot routing, at work-end — everywhere except a one-character difference in a symlink target. Git archaeology on the symlink itself was the only path to the root cause.
