# Decisions — casehub-langchain4j

## D1: Separate repo, not a fork or module in an existing repo

**Choice:** New `casehubio/casehub-langchain4j` repo
**Alternatives:**
- Fork of quarkus-langchain4j — they won't accept CaseHub integrations back upstream
- Module in platform — langchain4j interop is a distinct concern, not foundational infrastructure
- Module in blocks — blocks owns orchestration patterns, not cross-cutting enterprise concerns
**Rationale:** Single front door for LC4j enterprise enrichment. Clean dependency direction (depends on platform/neocortex/ledger, nothing depends on it). Matches casehub ecosystem pattern of focused repos per concern.
**Trade-offs:** One more repo to maintain, CI dispatch chain to wire.
**Sources:** ADR-0004 (dual-track strategy), parent/docs/new-repo-checklist.md
**Exploration:** quick
**Status:** captured

## D2: Per-SPI flat modules with naming convention

**Choice:** Flat sibling modules with `-core` / (bare) / `-spring` suffix convention
**Alternatives:**
- Nested module directories (core/quarkus/spring top-level groupings) — no existing casehub repo uses this pattern yet
- Single module per SPI (no framework split) — prevents pure-Java reuse
**Rationale:** Matches established casehub convention (platform: agent-langchain4j-core / agent-langchain4j / agent-langchain4j-spring). Consistency over novelty.
**Trade-offs:** Flat directory gets wide if many modules. Acceptable at 4 SPIs (12 modules).
**Sources:** platform module structure, neocortex module structure
**Exploration:** quick
**Status:** captured

## D3: Classpath activation with graceful degradation

**Choice:** CDI @DefaultBean / @Alternative @Priority (Quarkus), @ConditionalOnClass (Spring). No config required.
**Alternatives:**
- Config-gated (casehub.lc4j.audit.enabled=true) — adds friction, contradicts drop-in story
- Progressive (classpath basic, config advanced) — unnecessary complexity for decorator modules
**Rationale:** Same activation pattern platform already uses everywhere. Zero code change for the developer. If CaseHub deps are absent, module logs warning and no-ops.
**Trade-offs:** Possible CDI bean conflicts in complex deployments. Mitigated by priority tiers.
**Sources:** platform agent-langchain4j CDI tier design (2026-06-26 spec)
**Exploration:** quick
**Status:** captured

## D4: Both Quarkus and Spring from day one

**Choice:** Ship both framework variants in phase 1
**Alternatives:**
- Quarkus first, Spring follows — delays Spring community access
- Core only, framework wiring later — delays the classpath activation story
**Rationale:** CaseHub already has the dual-framework pattern proven across platform modules. The Spring community is large. Three-module pattern (core/quarkus/spring) is established.
**Trade-offs:** Doubles the testing surface. Accepted — platform already manages this.
**Sources:** platform agent-claude-core / agent-claude / agent-claude-spring pattern
**Exploration:** quick
**Status:** captured

## D5: Decorators over replacements — CaseHub is additive, not competitive

**Choice:** Lead modules are decorators that wrap existing LC4j implementations with enterprise concerns (audit, tenancy, governance). Do NOT replace LC4j's stores, retrievers, or providers.
**Alternatives:**
- Replace LC4j implementations with neocortex-backed versions — looks predatory/hostile, duplicates upstream work
- Both decorators and replacements equally — confuses the message
**Rationale:** The message is "we make langchain4j better for enterprise" not "we replace langchain4j with our stuff." Decorators work with ANY LC4j implementation the developer already chose. Additional implementations (hybrid search, SPLADE) are justified only where LC4j has acknowledged gaps (e.g. #4087) and the community would welcome them.
**Trade-offs:** Limits the surface area we can deliver. Accepted — defensibility and community trust matter more than breadth.
**Sources:** Adversarial review findings, LC4j community research (issues #4087, #4098, #1889, Quarkus WG discussion)
**Exploration:** deep-analysis (adversarial review + community research)
**Status:** captured

## D6: Four lead modules — audit, tenancy, governance, hybrid-search

**Choice:** Four modules for phase 1, all in the "safe" column per community research
**Alternatives:**
- Seven modules (one per LC4j SPI) — 5 of 7 are thin neocortex wrappers, looks like redoing LC4j
- Two modules (observability + agents only) — misses tenancy and hybrid search which are genuine gaps
- Three modules (decorators only, no hybrid-search) — hybrid search is explicitly welcomed (#4087)
**Rationale:**
- audit: LC4j does OTel, not cryptographic compliance evidence. No overlap.
- tenancy: LC4j has no framework-level tenancy SPI. Welcomed.
- governance: complementary to their OTel tracing. Human oversight gates are outside their scope.
- hybrid-search: open issue #4087, acknowledged gap, no LC4j solution. Welcomed.
**Trade-offs:** Smaller surface than originally planned. Additional modules can follow with documented justification (gap acknowledged, customer requested, different category).
**Sources:** LC4j issues #4087, #4098, #1889; PR #6303; Quarkus WG discussion; Microsoft partnership analysis
**Exploration:** deep-analysis (community research agent)
**Status:** captured

## D7: Move platform/agent-langchain4j to this repo

**Choice:** Absorb the existing ChatModel ↔ AgentProvider bridge from platform
**Alternatives:**
- Keep in platform as a dependency — the bridge is fundamentally langchain4j interop, not foundational infrastructure. Engine already depends on langchain4j-core directly.
**Rationale:** Consolidates all LC4j interop in one repo. Platform stays focused on foundation SPIs (Path, Preferences, Identity, AgentProvider SPI). The AgentProvider SPI stays in platform; the bridge that adapts ChatModel ↔ AgentProvider moves here.
**Trade-offs:** Engine/blocks add casehub-langchain4j as a dependency. Acceptable — they already depend on langchain4j-core.
**Sources:** platform agent-langchain4j-core, platform agent-langchain4j, platform agent-langchain4j-spring
**Exploration:** quick
**Status:** captured

## D8: Future SPI implementations require documented justification

**Choice:** Any module beyond the four lead modules must document why it's not predatory
**Alternatives:**
- Ship everything — risks looking hostile to the LC4j community
- Never ship implementations — misses genuine gaps where enterprise versions are welcomed
**Rationale:** Three valid justifications: (1) LC4j acknowledged the gap (cite issue), (2) it's a different category not a competing implementation, (3) a customer requested it. Each must be documented in the module's README.
**Trade-offs:** Slows expansion. Accepted — community trust is more valuable than speed.
**Sources:** Adversarial review, user direction ("shouldn't seem hostile or predatory")
**Exploration:** quick
**Status:** captured
