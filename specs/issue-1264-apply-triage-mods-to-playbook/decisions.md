## D1: Where to apply triage modifications

**Choice:** Handler-level modification with SPI for state lookup (Approach A)
**Alternatives:**
- Orchestrator-level via ExecutionMetadata — threads modifications through metadata, but pollutes a cross-cutting type with playbook-specific data
- Re-compilation from modified source — clean but requires storing original YAML, re-compilation cost, cross-repo change to CompilationResult
**Rationale:** Keeps changes within the engine, respects module boundaries via a thin SPI in common-core, and the Workflow tree traversal is bounded by the compiler's own step types. SourceMap on CompilationResult (already on PlaybookWorkerFunction) provides the mapping. No cross-repo changes needed.
**Trade-offs:** Handler grows in responsibility; SWF Workflow tree traversal is new code that must handle nested structures (blocks, try/catch, forEach, etc.)
**Sources:** FlowWorkerFunctionHandler.java:68-129, PlaybookWorkerFunction.java, PlaybookCompiler.java, BindingRecoveryState.java, CaseRecoveryStateRegistry.java
**Exploration:** quick
**Status:** captured

## D2: Cumulative modification semantics

**Choice:** Cumulative across all triage rounds — union of SkipSteps, last-writer-wins for ReplaceStepInput per stepId and UpdateParameter per key
**Alternatives:**
- Latest round only — simpler but forces TriageHandler to re-specify all prior modifications each time
**Rationale:** Matches the issue's "cumulatively" language. Gives triage maximum flexibility — each round can layer additional modifications without repeating prior ones.
**Trade-offs:** Merge logic is slightly more complex; if a TriageHandler wants to undo a prior modification, it currently can't (no "un-skip" operation). Acceptable for now — a future issue can add removal if needed.
**Sources:** Issue #1264 acceptance criteria, BindingRecoveryState.previousTriageModifications()
**Exploration:** quick
**Depends on:** D1 (modification approach)
**Status:** captured
