# Handoff — 2026-06-21

## What's Done

**S-scale batch: #527, #525, #523 — provider gap-fill, registry query, repository queries**

- engine#527 closed — wired all missing YAML-schema fields through ChatModelProviders: OpenAI (baseUrl, organizationId, frequencyPenalty, presencePenalty), Anthropic (baseUrl, version), Mistral (baseUrl). AgentConverter updated for all three.
- engine#525 closed — CaseDefinitionRegistry.findByIdentity(namespace, name, version) → Optional<CaseMetaModel>. Default method per SPI evolution protocol. Enables casehub-ops drift detection without catching RuntimeException.
- engine#523 closed — findByStatus/findAll/findByNamespaceAndName on CaseInstanceRepository. Default methods + InMemory + JPA implementations. Enables devtown/aml/clinical operational tooling.
- CLAUDE.md, DESIGN.md updated. Blog entry published.

## Immediate Next Step

Pick up new work. CI is green.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #543 | Migrate Worker primitives to casehub-worker-api | L | High | Major refactoring |
