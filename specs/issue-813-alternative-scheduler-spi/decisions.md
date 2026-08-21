## D1: Scope — both SPIs

**Choice:** db-scheduler replaces Quartz for both `JobScheduler` (timed/cron triggers) and `WorkerExecutionManager` (worker execution dispatch)
**Alternatives:**
- Worker execution only — smaller scope but consumers need both Quartz and db-scheduler on the classpath (split-brain)
- Worker execution first, JobScheduler later — incremental but temporary split-brain
**Rationale:** Clean single-scheduler default. No split-brain where one scheduler runs cron and another runs workers.
**Trade-offs:** Larger scope of work — must implement all `JobType` handlers for db-scheduler, not just worker execution
**Sources:** `common/spi/scheduler/JobScheduler.java`, `common/spi/scheduler/WorkerExecutionManager.java`
**Exploration:** quick
**Status:** captured

## D2: Job type dispatch — JobType enum

**Choice:** Replace `jobClass: Class<?>` on `ScheduledJobRequest` with a `JobType` enum
**Alternatives:**
- String-based type key — more extensible but loses compile-time exhaustiveness
- Handler registry pattern — CDI lookup by handler ID, most flexible but most complex
**Rationale:** The `jobClass` field is a Quartz leak (`Class<? extends org.quartz.Job>`). A sealed enum gives type-safe exhaustiveness checking. Each scheduler maps `JobType` to its own dispatch mechanism internally.
**Trade-offs:** Adding a new job type requires touching the enum — but engine job types are a closed set
**Sources:** `scheduler-quartz/QuartzJobScheduler.resolveJobClass()`, backup branch `JobType.java`
**Exploration:** quick
**Status:** captured

## D3: Schema management — scoped Flyway migration

**Choice:** Scoped Flyway migration at `db/engine-scheduler/migration/V1__Create_DbScheduler_Table.sql`
**Alternatives:**
- db-scheduler `createIfNotExists()` — zero migration files but schema invisible to Flyway, drift risk
- Hibernate auto-DDL — consistent with test approach but production Hibernate DDL is avoided
**Rationale:** Follows the engine-ledger pattern (`db/engine-ledger/migration/`). Consumers add `classpath:db/engine-scheduler/migration` to their Flyway locations. Explicit, auditable, consistent with platform conventions.
**Trade-offs:** Consumers must configure Flyway locations — but this is the established pattern
**Sources:** CLAUDE.md "No Migration Tooling" section, `persistence-hibernate/src/main/resources/db/migration/`
**Exploration:** quick
**Depends on:** D1 (db-scheduler needs a table only if it's implementing both SPIs)
**Status:** captured

## D4: Phasing — extract first, then add db-scheduler

**Choice:** Phase 1: extract orchestrators + fix jobClass leak. Phase 2: add db-scheduler module + make it default.
**Alternatives:**
- All together — fewer commits but harder to review, higher risk of tangled conflicts
**Rationale:** Each phase is independently reviewable. The extraction proves itself against Quartz before db-scheduler arrives. If the extraction breaks something, it's isolated.
**Trade-offs:** Two review cycles instead of one
**Sources:** backup branch `WorkerExecutionOrchestrator.java`, `RetryOrchestrator.java`
**Exploration:** quick
**Status:** captured
