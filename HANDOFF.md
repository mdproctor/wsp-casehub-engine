# Handoff — issue-377-distributed-per-agent-coordination

## This session

Designed, reviewed, and implemented the coordination vocabulary types (engine#1282).
Six types in `io.casehub.api.model.coordination`: TopologyType, DecisionScope,
DecisionType, StructuralRole, PlatformRole, AgentRole. Deep adversarial review
(4 dimensions, 57 issues, all resolved) shaped the final design — removed GRADUATED
and HIERARCHICAL from TopologyType, removed DecisionAuthority (derivable), deferred
TopologyDescriptor to #1283+. Settled CONTRACTOR naming (FIPA heritage, avoids
Worker-layer collision). Filed 11 new issues for gaps surfaced during review.

## Key decisions

- **Three-faceted AgentRole:** name (Path, open) + structuralRole (closed enum) + platformRole (closed enum, nullable)
- **CONTRACTOR not WORKER:** Worker is a platform execution primitive; coordination describes structural relationships
- **DecisionAuthority derived:** total function of TopologyType x StructuralRole — 6 valid, 6 invalid combinations
- **StructuralRole not RoleArchetype:** eidos owns "archetype" heavily (ArchetypeTerm, ArchetypeFamily, AgentDescriptor.archetype) for personality/capability profiles; StructuralRole avoids collision — these are the structural primitives that define topology shape
- **TopologyDescriptor deferred:** runtime state, not vocabulary — belongs with SPIs (#1283+)

## Next action

Engine#1283 (AgentCapabilityGuard) — the first consumer of the vocabulary types.
Read the updated spec at `specs/issue-1282-coordination-vocabulary-types/SPEC.md`
for the permission matrix, derivation mapping, and deferral decomposition table.

## Resolved — RoleArchetype → StructuralRole rename

Eidos owns "archetype" heavily: `ArchetypeTerm` (48 personality archetypes),
`ArchetypeFamily` (12 motivation quadrants), `AgentDescriptor.archetype` field.
All about *who the agent is* (capability/personality profile). The coordination
vocabulary's INITIATOR/PEER/CONTRACTOR describe *structural position* — completely
different concept. Renamed `RoleArchetype` → `StructuralRole` to avoid collision.
These are the structural primitives that define topology shape — SUPERVISOR is
structural *because* it requires INITIATOR, not independently.

## References

| Artifact | Path |
|----------|------|
| Spec (reviewed) | `specs/issue-1282-coordination-vocabulary-types/SPEC.md` |
| Journal | `JOURNAL.md` |
| Blog | `blog/2026-11-01-mdp01-the-vocabulary-that-shapes-everything.md` |
| Review workspaces | `~/reviews/casehub-slots/coordination-vocabulary-*-20261101-*/tracker.md` |
| Garden entry | `GE-20261101-0dd962` (coherence review blind spot) |
