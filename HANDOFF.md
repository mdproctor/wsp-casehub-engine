# Handoff — issue-212-yaml-helpdesk-apps

## What was finished

**casehubio/engine#1365 — YamlCaseHubDiscovery** (done, landing with work end)
- `YamlCaseHubDiscovery` in `runtime/bootstrap/` — startup observer at priority 2600 auto-creates CaseHub for YAML definitions without Java classes
- Extracted from `DefaultCaseDefinitionRegistry.registerCaseDefinition()`

**casehubio/engine#1366 — persistence-json module** (done, landing with work end)
- `casehub-engine-persistence-json` module: `CaseInstanceCache` backed by `JsonCollectionStore` (SQLite)
- `CaseInstanceDocument` for flat serialization; `PersistenceJsonBeans` CDI producer `@Alternative @Priority(200)`
- Scoped to single-user/low-concurrency (documented in Javadoc)
- Review fixes applied: null filter on `getAll()`, SPI return type, dead `CaseHubRuntime` injection removed

**casehubio/eidos#279 — call: agent worker function provider** (done, landing with work end)

**Squash review complete** — engine branch compacted from 5 to 2 commits. Artifacts promoted (2 specs, 1 blog).

## Active item

**casehubio/eidos#280 — Agent-task binding resolver [eidos]** — not started

## What is next in queue

- casehubio/casehub-pages#751 — Push topic REST publish endpoint [pages]
- casehubio/examples#220 — YAML helpdesk case lifecycle data + end-to-end validation [examples]

## Notes for next session

- Three done items landing: engine#1365, engine#1366, eidos#279
- `persistence-json` depends on `casehub-pages-data-json` — first cross-project dependency from engine to pages
- `SqliteConnectionFactory` test helper uses reflection — fragile if field names change
- Pre-existing test failure in runtime module: `SignalDedupExtendedTest` class resolution fails, blocks all runtime tests when run via surefire
