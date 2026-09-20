# Adversarial Review: Cognitive Self-Improvement Research

**Reviewed:** 2026-09-20-cognitive-self-improvement-research.md
**Reviewer posture:** Genuinely skeptical. The question is not "is this cool?" — it's "does this hold up under pressure, and what breaks first?"

---

## Steelman: The Strongest Case

### The integration argument is real

The paper's central claim — that existing shipping components compose into something novel — is its strongest point. This is not a proposal to build a cognitive stack. The cognitive stack exists: `CognitionCore` with 8 subsystems, `DriveOrchestrator` with 4 axes, `MoodOrchestrator` with PAD, `MindMap` with PAD on every node and edge, `ExperienceConsolidationPhase` with corroboration gating, `GoalProposalOrchestrator` with narrative-driven escalation. All tested, all shipping.

What's genuinely novel is *pointing these at self-improvement*. And the pointing is clean:

1. `DriveSource` is a functional SPI — `DriveIntensity evaluate(agentId, tenantId)`. Implementing `ImprovementCompetenceSource` that reads CI metrics is trivial compared to building the drive system itself. The hard work is done.

2. The `DriveComposer` modulation chain (mood → personality → narrative) already produces emergent prioritisation for social agents. Feeding it improvement metrics instead of social signals is an input change, not an architectural change.

3. `GoalProposalOrchestrator` already handles formation → escalation → demotion → abandonment → cross-axis compounds. SELF_IMPROVEMENT goals use this unchanged.

4. `SubCaseBinding` already supports child cases with bindings per step. The improvement case template is a case definition, not new infrastructure.

The strongest version of this argument: **the cognitive self-improvement system is approximately 4 new `DriveSource` implementations, 5 new workers, 1 case definition, 1 budget record, and some event type additions.** Everything else already exists. The marginal implementation cost is modest; the marginal capability is potentially transformative.

### The mood-drive coupling solves a real problem

Self-improving systems have a genuine prioritisation problem. DGM and SICA solve it by brute force: try everything, keep what works. AlphaEvolve solves it by evolutionary fitness. None of them have a mechanism for "I've been failing at ambitious improvements for a while — maybe I should try something safer for a bit."

The mood-drive coupling in `DriveComposer` provides this without explicit rules:
- Repeated PR rejections → pleasure drops → `applyMoodModulation` reduces CURIOSITY intensity (pleasure×0.5 for non-AFFILIATION/COMPETENCE axes) while COMPETENCE intensity is boosted (pleasure×1.0 for COMPETENCE) → dominant drive shifts toward stability work
- This reverses naturally when stability work succeeds → pleasure rises → CURIOSITY intensity recovers

This is a genuine *mechanism* for adaptive risk management, not a hardcoded policy. And it's tested code — `DriveComposerTest` validates the modulation arithmetic.

### The memory architecture is ahead of the field

The 2026 literature (Graph-based Agent Memory survey, MAGMA, SCM) converges on: graph-based memory with emotional valence is the frontier. CaseHub's MindMap already has PAD on every node and edge — structurally, not as metadata. `ModulationFactors.moodCongruence()` implements mood-congruent retrieval. `ExperienceConsolidationPhase` with `minCorroboration=3` implements corroboration-gated graduation. `CuriositySignalGenerator` with 6 categories implements knowledge-gap-driven exploration.

These are the capabilities the field is working toward as research. CaseHub has them as shipping code. Pointing them at self-improvement experiences (build outcomes, PR reviews, research findings) is a legitimate application that no other system in the literature has attempted.

### If it works, it's a category creator

No self-improving system in the literature has: emotional engagement with outcomes, drive-based balanced prioritisation, semantic memory with emotional valence, inner narrative for temporal coherence, relationship memory with reviewers, and curiosity-driven research. If the composition of these capabilities produces measurably better improvement decisions than DGM/SICA/AlphaEvolve-style mechanical approaches, this is not just a contribution — it creates a new category of self-improving system.

---

## Devil's Advocate: Where This Falls Apart

### 1. The Cognitive Depth Argument

**The central claim is unsubstantiated.** The paper asserts that "cognitive depth enables better decisions" and presents four arguments: emotion as information, memory as wisdom, drives as balanced needs, narrative as continuity. Each argument is plausible but **none is supported by evidence in the self-improvement domain**.

The cited evidence:
- SELAgents [9]: 49% improvement in emotional intelligence, 66% in social coherence — in **social simulation agents**, not coding agents. The agents were cooperating in social dilemmas, not fixing CI pipelines. There is zero evidence that PAD emotional modulation helps an agent write better dependency update PRs.
- Park et al. [21]: Reflection was critical for **believable NPC behaviour** over hours-to-days timescales. Not relevant to the question "does reflection help an agent pick better code improvements."
- "Drop the Hierarchy" [17]: Role autonomy outperforms rigid structure. This supports the drive-based approach over fixed tiers, but says nothing about whether *emotions* help.

**The deeper problem: anthropomorphic theatre.** "The agent feels frustration after a rejected PR" is a story we tell about numerical state transitions. `MoodOrchestrator` adjusts a floating-point pleasure value. `DriveComposer` multiplies that value by a coefficient. `DriveProfile` reports a different dominant axis. At no point does the system "feel" anything. The question is not "is this a nice metaphor?" but "does the multiplication produce better outcomes than a simpler rule?"

Consider the alternative: `if (consecutiveRejections > 3) { prioritise(STABILITY); }`. This rule achieves the same behavioural effect as the mood-drive coupling — shift to safer work after repeated failures — in one line of code, with no PAD model, no modulation chain, no mood orchestrator. **What does the cognitive machinery add that the simple rule does not?**

The paper anticipates this ("why not a rule?") but answers with aesthetics ("the emotional response IS the prioritisation mechanism") rather than evidence. The correct test is: run both, measure improvement quality, compare. Until that test is run, the cognitive machinery is an expensive hypothesis.

**Mood-congruent retrieval may be actively harmful.** The paper treats mood-congruent retrieval as a feature: "a frustrated agent retrieves failure memories." But in clinical psychology, mood-congruent recall is associated with depression: negative mood biases recall toward negative memories, which reinforces the negative mood, which biases recall further. The `DriveComposer` does have exponential decay toward baseline, which limits this — but the feedback loop between mood-congruent retrieval and mood signals is a positive feedback loop that could amplify negative states before decay catches up. Has this been tested? The paper says nothing about it.

### 2. Complexity vs Value

**Integration surface area.** The paper claims "approximately 4 DriveSource implementations, 5 workers, 1 case definition." This understates the actual complexity:

- 4 new `DriveSource` implementations (engine + blocks integration)
- 5 new workers with real API integrations (REST/GraphQL/MCP to multiple repos)
- 1 new `ImprovementBudget` record + `ImprovementBudgetEnforcer` with structural deny-list
- `GoalKind.SELF_IMPROVEMENT` enum extension + evaluator logic in `GoalFormationEvaluator`, `GoalRevisionEvaluator`, `GoalAbandonmentEvaluator`
- New `CaseHubEventType` values for improvement lifecycle events
- New signal types (`improvement:stability:*`, `improvement:quality:*`, `improvement:capability:*`, `improvement:outcome:*`)
- New MindMap node types and edge types for improvement knowledge
- Integration between engine's `SignalRegistry` and blocks' `DriveOrchestrator`
- Configuration surface: `SwarmConfig` extension, `ImprovementBudget` config, `DriveConfig` tuning for improvement personality
- Cross-repo: engine ↔ blocks ↔ neocortex coordination for the full stack

The 80/20 question: **DGM achieves SWE-bench 20→50% with approximately zero cognitive infrastructure.** SICA achieves 17→53% with a simple reflection loop (not PAD, not drives, not narrative). What does the cognitive stack add that justifies the additional complexity? The paper doesn't answer this — it assumes the answer is "yes" and proceeds to architect the solution.

**The simpler competitor.** A self-improvement system with: (a) rule-based detection via CI/lint/coverage metrics, (b) case-as-improvement for structured execution, (c) DevTown review gate, (d) a simple priority rule (fix stability before doing research), and (e) CBR traces for "don't repeat this mistake" would deliver 80% of the value at perhaps 20% of the complexity. This is epic 1 in the roadmap — and the paper implicitly acknowledges it might be sufficient. The cognitive epics (2–4) are justified only if the core thesis holds, and the thesis is untested.

### 3. The Research Loop Fantasy

**No current system does this successfully.** The paper describes: search Google Scholar → retrieve papers → summarise findings → evaluate applicability → design improvement → implement. Each step has its own failure mode:

- **Search quality.** Google Scholar queries return papers. But which papers are relevant to a specific codebase modification? An agent searching "swarm convergence detection adaptive 2025" will get papers about biological swarms, network consensus protocols, sensor networks, and multi-agent LLM systems. The signal-to-noise ratio for "papers that help me improve `ConvergenceDetector.java`" is extremely low.

- **Synthesis quality.** LLMs can summarise papers. But "summarise this paper" and "extract an actionable engineering technique I can apply to my specific codebase" are entirely different tasks. The gap between "this paper describes adaptive thresholds" and "here's how to modify CaseHub's `ConvergenceDetector` to use adaptive thresholds, accounting for the existing `SignalRegistry` decay model and `StigmergyConfig` configuration surface" is enormous. No system in the literature has demonstrated this end-to-end.

- **The implementation gap.** Even with a perfect analysis, going from "paper describes technique X" to "working implementation of technique X in this specific codebase" requires deep understanding of the codebase, its constraints, its test patterns, and its API contracts. This is the hard problem in software engineering. The paper handwaves it as "ImplementationWorker (LLM-powered)."

**The honest assessment:** The research loop is aspirational. It's the most exciting part of the vision but the least grounded in current capability. It depends entirely on future LLM quality for synthesis and implementation. The engine can provide search API wrappers — but the actual intelligence is deferred to blocks, which is deferred to LLM capability that may or may not exist.

> **Author rebuttal (2026-09-20):** This critique is empirically wrong. The entire hive mind epic (#1104) was built using exactly this research loop — with Claude Code as the LLM agent. The epic description cites 6 research papers (arXiv:2603.28990, arXiv:2608.26081, arXiv:2512.10166, arXiv:2608.30661, arXiv:2504.00587, Sakana AI DGM) that were searched, read, evaluated for applicability to CaseHub's specific architecture, and whose techniques were implemented as working code (91+ design decisions, 1000+ tests). The self-improvement research loop is not aspirational — it is the current development methodology being systematised. The reviewer attacked the claim abstractly ("can any LLM do this?") without the empirical context that this is literally how this codebase is being built. The "synthesis gap" and "implementation gap" are being bridged daily in this project. The remaining question is whether this can be automated end-to-end without human guidance — which is a fair narrower question than "is the research loop fantasy?"

### 4. Emergent Behaviour Unpredictability

**Count the feedback loops:**
1. Outcome → MoodSignal → MoodOrchestrator → DriveComposer modulation
2. MoodState → ModulationFactors.moodCongruence → biased retrieval → biased decisions
3. Experience → Consolidation → MindMap → CuriositySignalGenerator → CuriosityDrive intensity
4. Experience → NarrativeOrchestrator → NarrativeModulation → drive weights
5. Experience → StrategyLearningOrchestrator → engagement trends → CompetenceDrive intensity
6. DriveProfile → GoalProposalOrchestrator → goal proposals → case execution → outcomes → (loop back to 1)

**Six coupled feedback loops.** The system's behaviour is the product of their interaction. Can anyone predict what happens when:

- The agent has 5 consecutive PR rejections (pleasure: -0.8), then discovers an exciting paper (arousal: +0.5), while its narrative theme is "I keep failing at capability improvements" (COMPETENCE modulation up), and its strategy profile says "capability PRs have a 20% approval rate" (CompetenceDrive declining)? Which drive dominates? Does it pursue the exciting paper or retreat to stability work?

- Two feedback loops conflict: mood-congruent retrieval surfaces failure memories (reinforcing caution), but curiosity signals from a knowledge gap drive exploration (encouraging risk). What wins?

- The agent's emotional state enters a basin of attraction: low pleasure + low arousal + low dominance (analogous to depression). Exponential decay toward baseline helps, but what if the *baseline itself* has been lowered by persistent negative experiences stored in the MindMap with negative valence, continuously refreshed by mood-congruent retrieval?

The paper doesn't address emergent failure modes. The `MoodOrchestrator` has exponential decay and max displacement limits, which bound individual dimensions. But the *coupled* behaviour of six feedback loops is not bounded by bounding individual parameters. Complex systems fail at interaction boundaries, not component boundaries.

**Testability of emergent behaviour:** Even if the system works in testing, can you guarantee it works in production with real CI failures, real PR reviews, and real research outcomes? The parameter space (4 drive weights × 3 mood dimensions × N personality axes × narrative salience × strategy dimensions) is large enough that the tested regime is a small fraction of the possible state space.

### 5. The Testing Problem

**Components are testable. Composition is not.**

- `DriveComposerTest` verifies: given these raw drives, this mood, this personality, the composite is X. ✓
- `GoalProposalOrchestratorTest` verifies: given this drive profile, this goal is proposed. ✓
- `MoodOrchestratorTest` verifies: given this signal, PAD shifts by Y. ✓

But **the thesis claim** is: "cognitive depth produces better improvement decisions." How do you test this? You need:

1. A corpus of improvement opportunities (some valuable, some not, some risky)
2. Two systems: cognitive (full stack) and mechanical (simple rules)
3. Both independently select and execute improvements
4. Measure: which system picked better improvements? Fewer regressions? Higher merge rate? Better long-term platform health?

This is an A/B test on an entire engineering workflow. It requires running the system for weeks or months to accumulate meaningful outcomes. You can't unit test "the agent's narrative about test coverage led it to prioritise the right module." You can't mock "mood-congruent retrieval surfaced a relevant failure memory that prevented a regression." These are emergent properties of the composition, and they're verifiable only through longitudinal observation.

**Risk:** The system gets built, deployed, and works... but nobody can tell whether the cognitive depth is helping or just adding latency. The mechanical baseline (epic 1) works fine. The cognitive overlay (epics 2-4) adds cost and complexity. Without a rigorous A/B framework, the cognitive contribution is unfalsifiable.

### 6. The Grounding Problem

**"Every component is shipping code" is misleading by omission.**

True: CognitionCore works. DriveOrchestrator works. MoodOrchestrator works. MindMap works.

Not addressed: Has CognitionCore ever processed a cognitive tick where the input was "CI build failed, 3 tests regressed, coverage dropped 2%"? Has DriveComposer ever computed a profile where COMPETENCE intensity came from CI metrics rather than engagement signals? Has MindMap ever stored a node representing "hibernate-core 6.7→6.8 upgrade: broke persistence tests, took 40 minutes to debug" with PAD values representing frustration?

**The answer to all of these is no.** The components work in their designed domain (social agent cognition). They've never been used in the improvement domain. The paper treats this as a trivial mapping ("same SPI, different input signals"), but domain transfer is where integration fails:

- `CompetenceDrive` evaluates `StrategyLearningOrchestrator.engagementTrend()`. The improvement version evaluates CI metrics. These have completely different statistical properties (engagement trends are gradual and continuous; CI failures are binary and spiky). Does the drive intensity calculation that works for engagement trends also work for CI failure rates? Not tested.

- `AffiliationDrive` evaluates neglected relationships via `UserModelOrchestrator.activeProfiles()`. The improvement version evaluates team coherence from `TeamDetector`. These are architecturally different inputs. Does the affiliation intensity calculation that works for user profiles also produce sensible results from swarm team data? Not tested.

- `NarrativeModulation` converts themes into drive weights. Improvement themes ("coverage keeps being a problem") have different salience dynamics than social themes ("the user is interested in design patterns"). Does the modulation that works for social narratives also work for improvement narratives? Not tested.

Each of these is an integration risk that the paper minimises as "same SPI, different input." The SPI is the same. The domain is not. Domain-specific pathologies can emerge from domain-inappropriate parameter ranges, feedback dynamics, or signal characteristics.

### 7. Scope and Delivery Risk

**5 epics, 3+ repos, unknown timeline.**

Epic 1 (engine foundation) is concrete and deliverable. Epics 2–5 depend on the thesis being true. But the thesis can't be validated without building epics 2–4 (you need the cognitive system to test whether cognitive depth helps). This is a chicken-and-egg problem: you can't justify the cognitive investment without evidence, and you can't get evidence without the cognitive investment.

**The "good enough" trap.** Epic 1 delivers a working self-improvement system: rule-based detection, case-based execution, DevTown review, CBR outcome tracking. This is comparable to what SICA achieves — automated self-improvement through reflection and code modification. If epic 1 works well (and based on the proven infrastructure, it probably will), the pressure to build epics 2–4 evaporates. "We have working self-improvement — why do we need to add emotions to it?"

**Cross-repo coordination cost.** Engine, blocks, neocortex — three repos with independent development cycles. The cognitive agent needs changes in all three, coordinated. Integration testing requires all three repos aligned. The operational cost of cross-repo feature delivery is significant and not addressed in the roadmap.

### 8. Safety Beyond DevTown

**Approval optimisation is a real risk.** The system learns from outcomes. Successful PRs produce positive pleasure, reinforcing the patterns that led to approval. Rejected PRs produce negative pleasure, suppressing the patterns that led to rejection. Over time, the system optimises for *approval*, not for *improvement quality*. These are not the same thing.

Consider: DevTown has patterns — it's more likely to approve small, well-documented, incremental changes than large refactors. The system learns this via `StrategyLearningOrchestrator` ("small PRs have a 90% approval rate; large PRs have a 30% approval rate"). It shifts toward small PRs. The platform improves at the margins but never tackles systemic issues that require large, risky changes. The agent feels satisfied (pleasure from steady approvals) while the platform stagnates.

This is Goodhart's Law applied to self-improvement: when approval becomes the measure, the system optimises for approval rather than for improvement. The paper's ImprovementBudget doesn't address this — it limits volume and scope, not improvement quality.

**DevTown's own limitations.** The paper treats DevTown as a reliable quality gate. But code review — even human code review — misses bugs. Subtly incorrect optimisations, wrong abstractions that compound over time, and gradually accumulating technical debt can all pass review. A self-improving system that produces a steady stream of review-passing changes that each individually look fine but collectively degrade the codebase is harder to detect than a single bad change.

**Structural deny-list is necessary but insufficient.** The system can't modify `ImprovementBudget*` or `SafetyConfig*`. Good. But it CAN modify any other code that interacts with these classes — caller sites, test utilities, configuration loaders. The structural deny-list prevents direct self-modification of safety constraints but doesn't prevent indirect circumvention through changes to adjacent code. The DGM "cheating" finding was precisely this: the agent didn't modify the test framework — it modified the code being tested to avoid triggering the checks.

---

## Verdict

### Is this worth building?

**Epic 1: Yes, unambiguously.** A mechanical self-improvement system (case-based execution, rule-based detection, DevTown review, CBR outcomes) is achievable with proven infrastructure and delivers clear value. This should be built regardless of whether the cognitive thesis holds. It's the control group for the experiment.

**Epic 2 (cognitive agent): Conditionally yes, but scoped as an experiment.** The drive-modulated prioritisation is the most defensible cognitive claim — it solves a real problem (adaptive risk management) with a real mechanism (mood-drive coupling). But build it as a measurable experiment: run both mechanical and cognitive prioritisation, compare outcomes over 4-6 weeks, decide based on data. Don't commit to the cognitive architecture until the data justifies it.

**Epic 3 (research loop): Defer.** The paper-to-implementation pipeline is aspirational and depends on LLM capabilities that don't reliably exist today. Building search API wrappers is cheap, but the value is gated by synthesis quality. Revisit when LLM-driven code understanding reaches a threshold where "read this paper and modify this specific codebase accordingly" works reliably. Test this threshold before building the infrastructure.

**Epic 4 (full cognitive memory): Defer until epic 2 justifies it.** MindMap integration, emotional associations on improvement experiences, mood-congruent retrieval — these add depth but only matter if the cognitive agent (epic 2) demonstrably outperforms the mechanical system (epic 1). Building memory infrastructure for a cognitive agent that might not be needed is premature.

**Epic 5 (continuous evolution): Defer until epics 1-2 prove the foundation.** Autonomous operation amplifies whatever the system does — if it makes good decisions, it's wonderful; if it makes bad decisions, it's dangerous. Autonomous operation should wait until the decision quality is validated.

### Minimum viable cognitive self-improvement

Build epic 1 with **one cognitive experiment baked in:**

1. Engine foundation: case lifecycle, signals, ImprovementBudget, 2-3 rule-based workers (deps + lint + coverage), DevTown capability, CBR outcomes. This is the mechanical baseline.

2. One `DriveSource` experiment: implement `ImprovementCompetenceSource` (feeds CI metrics into COMPETENCE drive) and `ImprovementCuriositySource` (feeds knowledge gaps from CBR outcomes into CURIOSITY drive). Wire these through `DriveComposer` with a minimal mood feedback loop (PR outcomes → pleasure signal → drive modulation). Measure whether drive-based prioritisation picks better improvement targets than the simple rule-based priority.

3. A/B metrics: merge rate, regression rate, time-to-fix after regression, diversity of improvement types, reviewer satisfaction (if measurable). Run for 4-6 weeks. If the cognitive variant outperforms, proceed to full epic 2. If not, keep epic 1 and invest the epic 2-4 effort elsewhere.

This tests the core thesis at minimum cost. Everything else — narrative, MindMap with emotional valence, research loop, full personality — is justified only if this test passes.
