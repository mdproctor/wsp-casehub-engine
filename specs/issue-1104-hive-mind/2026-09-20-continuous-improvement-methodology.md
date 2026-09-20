# Continuous Improvement Methodology

**Purpose:** Structured methodology for autonomous self-improvement research,
analysis, and direction-setting. Grounded in established frameworks from
foresight studies, innovation management, and systematic review practice.

**Audience:** The self-improvement cognitive agent, and humans configuring it.

**Status:** Reference methodology for #1115 (Continuous Evolution Loop).
Governs how the system discovers, evaluates, and selects improvement
directions.

---

## 1. Foundational Frameworks

This methodology composes four established frameworks, each solving a
different part of the problem:

| Framework | Origin | What it solves |
|-----------|--------|---------------|
| **Horizon Scanning** | Foresight studies (OECD, EU JRC) | Detecting signals of change in the external landscape |
| **PRISMA Protocol** | Systematic review practice (Moher et al.) | Structured, reproducible research pipeline |
| **Technology Radar** | ThoughtWorks | Visualising and tracking technology maturity and adoption decisions |
| **Wardley Mapping** | Simon Wardley | Positioning components by evolution stage and value chain visibility |

The methodology also draws on innovation management taxonomy (Christensen,
Henderson & Clark, Ries) for selection strategy classification.

---

## 2. Signal Categories

External and internal signals are classified using standard horizon
scanning terminology:

| Category | Definition | Time horizon | Example |
|----------|-----------|-------------|---------|
| **Megatrend** | Large-scale, sustained force of change already reshaping the landscape | Years | "LLM agents are replacing static workflow orchestration" |
| **Weak signal** | Early, ambiguous indicator of potential change — not yet a clear trend | Months | "Two papers this quarter propose memory-augmented agent coordination" |
| **Wild card** | Low-probability, high-impact event that could dramatically reshape the landscape | Unpredictable | "Major framework open-sources a self-improving agent system" |

The system deposits signals with these categories as metadata. Megatrends
inform strategic direction. Weak signals trigger technology scouting.
Wild cards trigger immediate assessment.

---

## 3. Tiered Cadence

Research operates at three tiers of depth. Each tier uses the same PRISMA
pipeline (§4) but with different scope and thoroughness.

### Tier 1: Horizon Scan

**Scope:** All capability areas, all competitors, comprehensive research
survey.

**When:** Rare — initial baseline, quarterly refresh, major strategic pivot,
or when the selection strategy bias changes.

**Cost:** Large (hours of LLM time, many API calls).

**Output:** Complete Technology Radar (§5) and Wardley Map positions (§6).
Updates the Living Systematic Review corpus (§7).

**Process:**
1. Frame the scan using STEEP categories (Social, Technological,
   Environmental, Economic, Political) adapted for the platform's domain
2. Execute PRISMA pipeline at full depth across all capability areas
3. Build or rebuild the Technology Radar
4. Update Wardley Map positions
5. Produce gap analysis across all areas
6. Surface megatrends, weak signals, and wild cards

### Tier 2: Technology Scouting

**Scope:** One capability area, updated landscape for that area only.

**When:** Drive intensity shifts significantly for an area, an area's radar
section goes stale (freshness threshold crossed), or on configurable
schedule.

**Cost:** Medium (tens of minutes).

**Output:** Updated Technology Radar section for the area. Updated gap
analysis. New corpus entries.

**Process:**
1. Scope to the target capability area
2. Execute PRISMA pipeline at area depth — targeted queries, area-specific
   screening criteria
3. Update the area's Technology Radar blips
4. Refresh the area's Wardley Map positions
5. Produce area-specific gap analysis with impact/cost/ROI assessment

### Tier 3: Deep Dive

**Scope:** Specific technology blip, technique, competitor feature, or
paper's applicability within an already-mapped area.

**When:** Every improvement cycle involving capability growth. The common
case.

**Cost:** Small (minutes).

**Output:** Concrete improvement hypothesis or updated gap assessment. New
or updated corpus entry.

**Process:**
1. Query the Living Systematic Review corpus first — what do we already
   know?
2. Check for updates since last review (new papers, new versions, new
   competitor moves)
3. If new information found, execute PRISMA screening and extraction at
   focused depth
4. Confirm or revise the improvement hypothesis
5. Update corpus entry with new findings

---

## 4. Research Pipeline (PRISMA Protocol)

Each tier executes the same pipeline, adapted from the PRISMA 2020
protocol for systematic reviews. The pipeline is reproducible, auditable,
and its outputs are tracked in the Living Systematic Review corpus.

### Phase 1: Identification

**Standard term:** PRISMA Identification

Discover candidate sources across configured channels.

| Channel | Examples | Automation level |
|---------|---------|-----------------|
| Academic databases | arXiv, Google Scholar, Semantic Scholar | Structured API queries |
| Code repositories | GitHub trending, specific org repos | API search |
| Competitor documentation | Product docs, changelogs, blog posts | Web scraping / API |
| Conference proceedings | NeurIPS, ICML, AAAI, EMNLP | Database query |
| Industry reports | Gartner, ThoughtWorks Radar, CNCF surveys | Periodic fetch |
| Internal evidence | CBR traces, EventLog history, outcome records | Direct query |

**Output:** Candidate set — a list of sources to evaluate.

**Depth scaling:**
- Horizon Scan: broad multi-channel, exhaustive queries
- Technology Scouting: targeted to one area, known channels
- Deep Dive: corpus-first, check one or two channels for updates

### Phase 2: Screening

**Standard term:** PRISMA Screening

Filter candidates against inclusion/exclusion criteria via title and
abstract review.

| Criterion | Screening question |
|-----------|--------------------|
| Relevance | Does this address a capability area the system cares about? |
| Applicability | Is this technique compatible with the platform's architecture? |
| Maturity | Is this empirically validated or purely theoretical? |
| Recency | Is this current enough to be actionable? |
| Novelty | Does the corpus already contain this or substantially similar work? |

**Output:** Screened set — sources that pass inclusion criteria.

**HIL gate:** Sources that cannot be retrieved (paywalled, institutional
access) are added to the HIL queue (§8) rather than silently dropped.

### Phase 3: Eligibility (Full-Text Assessment)

**Standard term:** PRISMA Eligibility

Deeper reading of each screened source to confirm it meets criteria.
Assesses methodological quality and relevance to specific gaps.

**Output:** Included set — sources confirmed as relevant and actionable.

### Phase 4: Data Extraction

**Standard term:** PRISMA Data Extraction

From each included source, extract structured findings:

| Field | What to extract |
|-------|----------------|
| Key technique | The core method or approach described |
| Claimed benefits | What the source claims it achieves |
| Evidence quality | Empirical validation, benchmarks, production use |
| Limitations | Acknowledged or inferred constraints |
| Applicability assessment | How well this maps to our architecture |
| Implementation complexity | Estimated effort to adopt |
| Capability area mapping | Which area(s) this finding applies to |

**Output:** Extraction records stored in the Living Systematic Review
corpus with summaries.

### Phase 5: Synthesis

**Standard term:** PRISMA Synthesis (narrative synthesis, not meta-analysis)

Group related findings into themes. Identify patterns, contradictions,
and convergences across sources.

**Synthesis operations:**
- **Thematic grouping** — "Papers A, C, and F all propose variations of
  adaptive signal decay. Common principle: X."
- **Contradiction detection** — "Paper B claims technique Y improves
  latency; Paper D found no significant effect under load."
- **Convergence identification** — "Three independent sources (academic,
  open-source, competitor) validate approach Z."
- **Gap identification** — "No source addresses the interaction between
  technique X and our specific constraint Y."

### Phase 6: Triangulation

**Not in standard PRISMA — added for multi-source validation.**

Cross-reference synthesised themes against multiple evidence types:

| Evidence type | What it provides |
|--------------|-----------------|
| Academic research | Theoretical foundation, empirical validation |
| Industry practice | Production-proven at scale |
| Open-source implementations | Concrete code, community validation |
| Competitor adoption | Market validation, competitive pressure signal |
| Internal CBR traces | Our own historical experience with similar approaches |

A finding validated by 3+ evidence types is high confidence. A finding
from a single source type is low confidence and needs further scouting.

### Phase 7: Prioritisation

Rank synthesised themes using a structured assessment per capability area:

| Dimension | Question | Scale |
|-----------|----------|-------|
| **Impact** | How much would this improve the platform? | Low / Medium / High / Transformative |
| **Cost** | How expensive to implement? (effort, risk, dependencies) | Low / Medium / High / Prohibitive |
| **ROI** | Impact relative to cost | Computed |
| **Feasibility** | Can we implement this given current architecture? | Ready / Adaptation needed / Major rework / Blocked |
| **Alignment** | Does this fit our design principles? | Strong / Moderate / Weak / Conflicting |
| **Confidence** | How well-validated is the finding? | High (3+ evidence types) / Medium / Low |

### Phase 8: Hypothesis Formation

Convert top-priority themes into concrete improvement hypotheses:

```
Hypothesis: Applying [technique] from [source] to [component] would
improve [metric] by approximately [estimate], with implementation
complexity [assessment].

Evidence: [triangulation summary]
Risk: [what could go wrong]
Prerequisite: [dependencies]
Capability area: [area]
Technology Radar recommendation: [Adopt / Trial / Assess]
```

Hypotheses become inputs to the improvement goal formation pipeline
(D92, D106).

---

## 5. Technology Radar

The persistent artifact that tracks the maturity and adoption status of
every technology blip the system has evaluated. Structure follows the
ThoughtWorks Technology Radar model.

### Quadrants (capability-area clusters)

Quadrants group related capability areas. The initial set:

| Quadrant | Capability areas |
|----------|-----------------|
| **Platform** | Stability, Performance, Execution, Integration |
| **Coordination** | Coordination, Perception, Autonomy |
| **Cognition** | Cognitive reasoning, Cognitive memory |
| **Governance** | Safety, compliance, trust |

Quadrants evolve as the capability taxonomy evolves (D109).

### Rings (adoption status)

| Ring | Meaning | Action |
|------|---------|--------|
| **Adopt** | Proven, ready for production use. The system should actively pursue improvements in this area using this technique. | Execute |
| **Trial** | Mature enough for pilot. Early adoption can capture advantage. Run a bounded improvement case to validate. | Pilot |
| **Assess** | Shows promise but unproven at our scale. Track developments, begin internal experimentation. | Investigate |
| **Hold** | Known limitations or risks. Do not use for new improvements. May be a technique we tried and found wanting. | Avoid |

### Blip lifecycle

A blip (a specific technology or technique) enters the radar at **Assess**
when first identified by screening. It moves inward through **Trial** →
**Adopt** based on evidence accumulation and pilot outcomes. It moves to
**Hold** when evidence shows it doesn't work for this platform, or when
a better alternative supersedes it.

Blip movements are tracked with timestamps and rationale — the radar
has a change log.

---

## 6. Wardley Map Positioning

Each component in the platform's value chain is positioned on two axes:

- **Y-axis (visibility):** How visible is this component to the end user?
  Ranges from invisible infrastructure to direct user value.
- **X-axis (evolution):** What stage of evolution is this component at?

| Evolution stage | Characteristics | Strategic implication |
|----------------|----------------|---------------------|
| **Genesis** | Novel, poorly understood, requires exploration | Pioneering innovation |
| **Custom** | Understood but requires bespoke implementation | Architectural innovation |
| **Product** | Increasingly standardised, multiple implementations exist | Sustaining innovation |
| **Commodity** | Well-defined, interchangeable, utility | Operational efficiency |

The fit-gap analysis (§4.7) uses Wardley positions to identify where the
platform is building custom solutions for commodity problems (waste) or
where commodity components are treated as genesis (under-investment).

---

## 7. Living Systematic Review (Research Corpus)

The persistent, searchable store of all research findings. Named after
the "Living Systematic Review" methodology — a systematic review that is
continuously updated as new evidence emerges, rather than being a
point-in-time snapshot.

### Corpus structure

| Component | Contents | Purpose |
|-----------|----------|---------|
| **Source records** | Full text or structured extract of each paper/article/doc | Avoid re-downloading |
| **Extraction summaries** | Per-source structured findings (Phase 4 output) | Avoid re-analyzing — the indexed unit for future retrieval |
| **Metadata** | URL, authors, date, retrieval date, freshness, provenance | Searchability and staleness |
| **Cross-references** | Links between sources addressing the same technique | Supports synthesis |
| **PRISMA flow records** | Which scan identified this source, at what tier, screening decisions | Reproducibility and audit |

### Index

Searchable by capability area, technique, keyword, date range, evidence
type, and confidence level. The index includes extraction summaries — not
just titles — so the system can determine relevance without re-reading
full sources.

### Freshness model

| Source type | Staleness threshold | Rationale |
|-------------|--------------------|-----------| 
| Academic papers | 12 months | Rarely go stale; findings are durable |
| Competitor feature docs | 3 months | Change frequently with releases |
| Open-source repos | 6 months | Active repos evolve; abandoned repos don't |
| Industry reports | 12 months | Annual or quarterly publication cycle |
| Internal CBR traces | Never stale | Our own evidence, always current |

Stale entries are flagged for re-check during the next technology
scouting pass for their capability area.

---

## 8. HIL Queue (Human-in-the-Loop)

Papers behind paywalls, institutional access walls, or requiring manual
retrieval are queued rather than silently skipped.

### Queue entry

| Field | Content |
|-------|---------|
| Source reference | URL, DOI, citation |
| Reason needed | Which capability area, which gap, what the system expects to learn |
| Priority | Based on how many research paths reference it |
| Blocking | Which hypotheses or assessments are waiting on this source |
| Date queued | When the system first identified the need |

### Queue lifecycle

1. Source identified during PRISMA Screening as relevant but inaccessible
2. Entry added to HIL queue with priority and blocking information
3. Signal emitted: `improvement:research:hil-needed`
4. Human retrieves and provides the source
5. Source enters corpus, triggers re-evaluation of blocked paths
6. Queue entry marked resolved

Priority is dynamic — if multiple scans reference the same source, its
priority increases. Sources blocking high-impact hypotheses are surfaced
more prominently.

---

## 9. Selection Strategies

How the system chooses where to focus, given the Technology Radar state,
gap analysis, and drive profile. Strategies use established innovation
management terminology.

| Strategy | Innovation type | Focus | Drive alignment |
|----------|----------------|-------|-----------------|
| **Incremental** | Incremental innovation | Quick wins, close obvious gaps, immediate value | High COMPETENCE — "fix what's broken" |
| **Sustaining** | Sustaining innovation (Christensen) | Refine existing capabilities to excellence along current trajectory | Moderate COMPETENCE — "make what works great" |
| **Architectural** | Architectural innovation (Henderson & Clark) | Restructure how components fit together for long-term capability | High AUTONOMY — "reshape the foundation" |
| **Radical** | Radical/breakthrough innovation | Build what nobody has, explore the frontier | High CURIOSITY — "pioneer new territory" |
| **Pivot** | Strategic pivot (Ries) | Redirect effort from a direction that isn't working | Triggered by negative outcome patterns in CBR |

### Strategy emergence

The active strategy **emerges from the drive profile** rather than being
explicitly selected:

1. Drive system evaluates all axes (COMPETENCE, CURIOSITY, AUTONOMY,
   AFFILIATION)
2. Dominant drive maps to a natural strategy bias (table above)
3. Strategy bias weights the prioritisation phase (§4.7) — incremental
   strategy weights cost/feasibility higher; radical strategy weights
   impact/novelty higher
4. Human can set a **strategy override** ("focus on sustaining this
   quarter") that modulates drive weights without suppressing them

### Strategy transitions

The system should not oscillate between strategies. A minimum dwell time
(configurable, default: 2 weeks) prevents strategy thrashing. Strategy
transitions are logged in EventLog and trigger a technology scouting
pass for the affected capability areas.

---

## 10. Capability Area Taxonomy

The dimensions across which the system evaluates itself. The taxonomy is
a living model — the initial set is a bootstrap that evolves through
landscape analysis and self-assessment.

### Bootstrap taxonomy

| Area | Covers | Typical metrics |
|------|--------|----------------|
| Stability | CI, tests, build reliability, error rates | CI pass rate, flaky test count, build time |
| Performance | Latency, throughput, resource usage | p50/p95/p99 latency, throughput, memory |
| Execution | Case lifecycle, worker dispatch, routing | Case completion rate, dispatch latency, routing accuracy |
| Coordination | Stigmergy, swarm, team formation | Convergence time, coordination overhead, team coherence |
| Perception | Observation, signal detection, environment sensing | Signal-to-noise ratio, detection latency, coverage |
| Autonomy | Self-provisioning, self-improvement, self-direction | Autonomous resolution rate, intervention frequency |
| Cognitive reasoning | Strategy, decision-making, planning, goal formation | Decision quality, plan success rate, goal completion |
| Cognitive memory | Knowledge storage, retrieval, consolidation, learning | Retrieval precision, consolidation rate, knowledge freshness |
| Safety | Trust, budget enforcement, guardrails, compliance | Trust violation rate, budget adherence, safety incident count |
| Integration | API surface, cross-repo coherence, ecosystem | API coverage, breaking change rate, compatibility score |

### Taxonomy evolution

The taxonomy is not fixed. The system can:

- **Discover** new areas through landscape analysis ("competitors
  distinguish 'short-term memory' from 'working memory' as separate
  concerns — our 'cognitive memory' should split")
- **Merge** areas that turn out to be the same concern
- **Split** areas that are too broad for meaningful assessment
- **Rename** areas as terminology evolves
- **Deprecate** areas that become irrelevant

Taxonomy changes require consensus (multiple signals from different
research tiers) and produce an EventLog entry. The Technology Radar
quadrants (§5) adapt when the taxonomy changes.

---

## 11. Process Flow

The complete methodology flow for a single evaluation cycle:

```
Drive profile → dominant drive → strategy bias
       │
       ▼
Capability area assessment → which areas need attention?
       │
       ├─ Area stale? ──────────── Tier 2: Technology Scouting
       ├─ No radar at all? ─────── Tier 1: Horizon Scan
       └─ Area current ─────────── Tier 3: Deep Dive
              │
              ▼
       PRISMA Pipeline (at appropriate depth)
       Identification → Screening → Eligibility →
       Extraction → Synthesis → Triangulation →
       Prioritisation → Hypothesis Formation
              │
              ▼
       Update Technology Radar blips
       Update Wardley Map positions
       Store findings in Living Systematic Review
              │
              ▼
       Improvement hypotheses → Goal formation pipeline
```

---

## 12. Terminology Reference

Quick reference for standard terms used throughout this methodology and
the #1115 implementation.

| Term | Source | Meaning in this context |
|------|--------|------------------------|
| Horizon scan | Foresight studies | Comprehensive landscape analysis across all capability areas |
| Technology scouting | Innovation management | Targeted investigation of one capability area |
| Deep dive | Common usage | Focused investigation of a specific technique or blip |
| Blip | ThoughtWorks Radar | A specific technology or technique tracked on the radar |
| PRISMA protocol | Systematic review | The structured research pipeline phases |
| Screening | PRISMA | Filtering candidates against inclusion/exclusion criteria |
| Extraction | PRISMA | Pulling structured findings from each included source |
| Synthesis | PRISMA | Grouping findings into themes |
| Triangulation | Research methodology | Cross-referencing themes against multiple evidence types |
| Living systematic review | Evidence-based practice | Continuously updated corpus (not point-in-time snapshot) |
| Megatrend | Horizon scanning | Large-scale sustained force of change |
| Weak signal | Horizon scanning | Early ambiguous indicator of potential change |
| Wild card | Horizon scanning | Low-probability high-impact event |
| Adopt / Trial / Assess / Hold | Technology Radar | Maturity rings indicating adoption recommendation |
| Genesis / Custom / Product / Commodity | Wardley Mapping | Evolution stages of a component |
| Incremental innovation | Innovation management | Quick wins, close obvious gaps |
| Sustaining innovation | Christensen | Refine along current trajectory |
| Architectural innovation | Henderson & Clark | Restructure how components fit together |
| Radical innovation | Innovation management | Build what nobody has |
| Strategic pivot | Ries (Lean Startup) | Redirect effort from a failing direction |
| HIL | Common | Human-in-the-loop — manual intervention required |
| Gap analysis | Strategy | Current state vs desired state assessment |
| ROI | Finance | Return on investment — impact relative to cost |
| STEEP | Foresight | Social, Technological, Environmental, Economic, Political scanning categories |

---

## References

- Kitchenham, B. & Charters, S. (2007). Guidelines for performing Systematic Literature Reviews in Software Engineering.
- Moher, D. et al. (2009). PRISMA: Preferred Reporting Items for Systematic Reviews and Meta-Analyses.
- Page, M. et al. (2021). PRISMA 2020 statement: an updated guideline for reporting systematic reviews.
- Wardley, S. (2016). Wardley Maps (CC BY-SA 4.0). https://wardleymaps.com
- ThoughtWorks Technology Radar. https://www.thoughtworks.com/radar
- Christensen, C. (1997). The Innovator's Dilemma.
- Henderson, R. & Clark, K. (1990). Architectural Innovation.
- Ries, E. (2011). The Lean Startup.
- Cuhls, K. (2020). Horizon Scanning in Foresight. Futures & Foresight Science.
- OECD. Horizon Scanning methodology.
- RAND Corporation. Technology Radar: Methodology (2008).
- arXiv:2507.21046. Self-Evolving Agents Survey.
