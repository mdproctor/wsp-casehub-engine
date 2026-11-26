# Handoff — issue-212-yaml-helpdesk-apps

## What was finished this session

**casehubio/engine#1365 — YamlCaseHubDiscovery** (done on branch)
- New `YamlCaseHubDiscovery` in `runtime/bootstrap/` — startup observer at priority 2600 that creates CaseHub instances for YAML definitions without explicit Java classes
- Extracted anonymous CaseHub creation from `DefaultCaseDefinitionRegistry.registerCaseDefinition()` into the dedicated discovery component
- 4 unit tests, 1567 runtime module tests green
- Issue specified `engine/flow` but we placed it in `runtime` because `flow` can't see `CaseHubRegistry`

**casehubio/engine#1366 — persistence-json module** (done on branch)
- New `casehub-engine-persistence-json` module implementing `CaseInstanceCache` backed by `JsonCollectionStore` (SQLite)
- `CaseInstanceDocument` record for flat serialization; reconstructs `CaseMetaModel` from `CaseDefinitionRegistry` on retrieval
- `PersistenceJsonBeans` CDI producer returns `CaseInstanceCache` (SPI interface), `@Alternative @Priority(200)`
- 6 unit tests, full project build green
- Review fix: null filter on `getAll()`, SPI return type on producer

## What is next

**casehubio/eidos#279 — call: agent worker function provider [eidos]**
- Different repo (eidos, not engine)
- Part of the same YAML helpdesk plan
- Queue position: 5/14

## Notes for next session

- Both #1365 and #1366 are marked done on the branch but not yet landed on main — they await `work end`
- The `persistence-json` module depends on `casehub-pages-data-json` — first cross-project dependency from engine to pages
- `SqliteConnectionFactory` test helper uses reflection to set CDI-injected fields — fragile if field names change, but adequate for now
- IntelliJ was flaky this session (timeout on open, IDE freeze requiring restart) — may need to be re-opened at start of next session
