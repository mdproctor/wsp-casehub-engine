# Handoff — 2026-06-28

## What's Done

**#565: re-publish casehub-engine-api SNAPSHOT — CLOSED**

Root cause was two-fold: (1) `casehub-worker` repo was private — its Maven packages were invisible to other repos' `GITHUB_TOKEN`, breaking CI dependency resolution since June 23; (2) `WorkItemStatus`/`WorkItemCreateRequest` moved packages in casehub-work#275, engine imports were stale. Also pushed unpushed `casehub-worker` commit making `WorkerFunction` a marker interface, and fixed pre-existing `HumanTaskPlannerIntegrationTest` failure (`TenantScopedPrincipal` @RequestScoped in @ConsumeEvent context). All casehubio repos now public. CI green, SNAPSHOT publishing on merge.

## What's Left

- Consumer repo import migration (aml#69, devtown#96) · S · Low each

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
