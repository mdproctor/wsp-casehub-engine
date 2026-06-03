---
layout: post
title: "Four issues, two architectural surprises"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [rls, postgresql, hibernate-reactive, cdi, tenancy]
---

The tenancy enforcement batch took longer than the individual issue sizes suggested. Four issues on one branch: a migration (#411), a CDI qualifier pattern (#405), DB-level Row Level Security (#406), and a registry hashCode bug (#410). The migration was mechanical. The other three had opinions.

## The qualifier pattern

Engine#405 asked for `@CrossTenant` as an access-control gate on cross-tenant SPIs. The qualifier itself was two annotations and a producer — straightforward. The wrinkle: recovery services need cross-tenant access but run outside any request context. Production `CurrentPrincipal` is `@RequestScoped`. Calling it from an `@ApplicationScoped` recovery service outside a request throws `ContextNotActiveException`.

The spec called for a system-actor principal that always returns `isCrossTenantAdmin() = true`. Platform doesn't have one yet, so I added `SystemCurrentPrincipal` to the engine — `@ApplicationScoped @EngineSystem`, not `@DefaultBean`, selected only via the `@EngineSystem` qualifier in the producer. When platform ships the real thing, one injection point changes and this class disappears.

## RLS: two surprises and a redesign

Engine#406 started with a CDI interceptor approach — annotate repository classes with `@TenantBound`, intercept calls, inject `SET LOCAL "casehub.tenancy_id"`. Clean on paper.

First surprise: PostgreSQL `SET LOCAL` doesn't accept JDBC bind parameters. `session.createNativeQuery("SET LOCAL ... = :tid").setParameter("tid", value)` compiles fine and fails at runtime. PostgreSQL treats configuration statements differently from DML — no prepared statement protocol, no `$1`. String interpolation with a sanitization guard is the only option.

Second surprise: CDI interceptors don't work for reactive code. An `@AroundInvoke` interceptor runs synchronously and completes before the `Uni` pipeline starts executing. By the time Hibernate Reactive opens a connection and begins the reactive session, the interceptor is long gone — and so is any `SET LOCAL` it tried to issue. The SET goes to a different connection entirely.

I brought Claude in to rethink the approach. We landed on a `TenantAwareRepository` base class that wraps `Panache.withTransaction()` and injects `SET LOCAL` inside the reactive pipeline, before the work runs. Reads also go through `withTenantTransaction()` rather than `withSession()` — `SET LOCAL` only applies within an explicit transaction, not in autocommit mode.

Cross-tenant access uses `SET LOCAL ROLE casehub_crosstenancy` (a BYPASSRLS PostgreSQL role) rather than a second policy branch. Simpler policy, auditable in `pg_roles`.

Two bugs surfaced during implementation. The `TABLES` list in `RlsPolicyApplicator` had `sub_case_group` — the entity's `@Table` annotation says `subcase_group`. And the BYPASSRLS role was granted correctly but never got `SELECT/INSERT/UPDATE/DELETE` on the engine tables, so every cross-tenant query failed with `permission denied`. Both caught before any commit reached main.

One limitation the integration test exposed: Quarkus Dev Services creates a PostgreSQL superuser, and superusers bypass `FORCE ROW LEVEL SECURITY` unconditionally. The test confirms the BYPASSRLS path and application-level filtering work — kernel-level RLS enforcement against the policy requires a non-superuser DB user to test.

## The registry fix

Engine#410 was a mutable hashCode key in a `ConcurrentHashMap<CaseMetaModel, CaseDefinition>`. `CaseMetaModel` has public setters on `namespace`, `name`, `version` — the three fields used in `hashCode()`. If any of those change after a `put()`, `Map.get()` looks in the wrong bucket. There was a defensive linear scan already in place as a guard; the fix was to replace the map key with an immutable `CaseKey` record. Two maps became one `Map<CaseKey, RegistryEntry>` — one atomic `put()`, no consistency window.
