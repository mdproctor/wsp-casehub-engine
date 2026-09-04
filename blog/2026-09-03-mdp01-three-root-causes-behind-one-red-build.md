---
layout: post
title: "Three Root Causes Behind One Red Build"
date: 2026-09-03
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [ci, neocortex, api-compatibility, maven]
---

# Three Root Causes Behind One Red Build

CI was red with a single symptom: `Non-parseable POM`. Maven couldn't even start. The fix looked trivial — a stray `-->` XML comment closer left behind when example modules were uncommented across a series of commits. One commit removed the opening `<!-- temporarily excluded...` but left the closing `-->` dangling at line 141. Maven treated everything between that orphaned closer and the next XML tag as text content, and refused to parse the file.

That got the POM parsing again, but compilation failed immediately on three files. The neocortex dependency — a `0.2-SNAPSHOT` — had changed its API underneath us. The `Confidence` type replaced raw `double` for confidence values across `CbrCase`, `Outcome`, `MemoryInput`, and `Memory`. The `Outcome` record gained an `Instant timestamp` field and renamed `importance()` to `confidence()`. `MemoryInput` and `Memory` each gained three PAD (pleasure/arousal/dominance) fields. Every constructor call site in the engine needed updating — not just the three production files, but eight test files constructing those records with the old signatures.

The third issue was subtler. `Binding.Builder` had gained an `on(String)` convenience overload alongside the existing `on(Trigger)`. One test called `on(null)` to verify null rejection — now ambiguous because `null` matches both parameter types. A cast to `(Trigger)` resolved it.

Seventeen files changed in total. The interesting pattern: the POM break masked everything else. CI couldn't even reach compilation, so the API incompatibilities had been accumulating silently since the neocortex SNAPSHOT was published. A parseable POM and a non-parseable POM produce the same CI status — red — but the blast radius behind that status was completely different.
