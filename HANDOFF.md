# Session Handover — 2026-11-23

**Branch:** `issue-1295-detected-role-bridging-spi`
**Slot:** 228 (engine + agency + eidos)
**Issue:** casehubio/engine#1295

## What happened

Reopened #1295 was actually complete in both engine (SPI on main) and agency (bridging on main). The session redesigned the architecture seam: moved `RoleBridgingPolicy` from agency to eidos as `RoleChangePolicy`, added `RoleDeclaration` and `RoleProfile` to `AgentDescriptor` — establishing the declared-vs-operable role model that parallels capabilities.

## Decisions

- `RoleChangePolicy` (not `RoleEvolutionPolicy`) — "evolution" is a term of art for improvement-directed change; role governance is change management
- Policy lives in eidos-api, not agency-core or engine-api — governance rules belong with identity layer
- `BridgedRoleState` stays in agency — orchestration runtime state, not identity governance
- Engine unchanged — SPI types (`RoleBridgingStrategy`, `RoleSuggestion`) correctly placed

## Cross-repo state

| Repo | Branch commits | Status |
|------|---------------|--------|
| eidos | 3 (types + AgentDescriptor + cosmetic fix) | needs landing |
| agency | 1 (delete old, update imports) | needs landing |
| engine | 0 (stamp only) | close stamp on branch |

## Follow-on issues (eidos)

- #268 — RoleDeclaration max length (XS)
- #269 — RoleProfile max count (XS)
- #270 — JPA persistence for roleProfile (S)
- #271 — RoleHealth probe (M/High)

## References

| Artifact | Path |
|----------|------|
| Blog | `blog/2026-11-23-mdp02-roles-belong-with-identity.md` |
| Design spec | `proj/docs/specs/issue-1295-detected-role-agent-role-bridging/` |
| Plan | `proj/plans/2026-11-07-detected-role-agent-role-bridging.md` |
