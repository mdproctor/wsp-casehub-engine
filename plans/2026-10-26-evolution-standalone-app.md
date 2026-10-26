# Evolution Standalone App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1225 — Restructure evolution conductor as standalone app within devtown
**Issue group:** #1225

**Goal:** Mount the evolution conductor as a standalone page at `/evolution` in devtown, separate from the review dashboard, and wire cross-app integration between the two experiences.

**Architecture:** The evolution workbench (already built as `@casehubio/mosaic-evolution-workbench`) becomes a full-page experience at `/evolution` instead of a tab in the review dashboard. Client-side path routing in `index.ts` selects which experience to mount. The backend Java code (API, bootstrap, capability areas) is already implemented — this plan addresses frontend wiring, a bootstrap fix, and cross-app integration (conductor-origin indicator, deep links).

**Tech Stack:** TypeScript/Lit (frontend), Java/Quarkus (backend), pages-ui DSL, esbuild

## Global Constraints

- **Repos:** all changes in `casehubio/devtown` (app module). No engine changes expected.
- **packages-ui DSL:** use `page()`, `tabs()`, `hostPanel()`, `registerPanel()` from `@casehubio/pages-ui` and `@casehubio/pages-runtime`
- **Evolution custom element:** `agency-evolution-workbench` from `@casehubio/mosaic-evolution-workbench`
- **SPA routing:** `quarkus.quinoa.enable-spa-routing=true` — server returns `index.html` for all unmatched paths; client-side routing selects the view
- **Existing backend:** `DevtownEvolutionResource` at `/api/devtown/evolution` is fully implemented (state, inbox, streams, deny/watch patterns, gate policy, circuit breaker endpoints)
- **Design principle:** evolution reads review's event log; review doesn't know evolution exists (one-way dependency). The only review-side intrusion is a small conductor-origin indicator on conductor-driven PRs.

---

## Batch 1: Standalone Evolution Page

After this batch, navigating to `/evolution` shows the evolution workbench as a full-page experience. The review dashboard at `/` is unchanged.

### Task 1: Wire evolution packages and create standalone page routing

**Files:**
- Modify: `devtown/app/src/main/webui/package.json` — add evolution-workbench and evolution-config dependencies
- Create: `devtown/app/src/main/webui/src/views/evolution.ts` — evolution page view
- Modify: `devtown/app/src/main/webui/src/index.ts` — add client-side path routing for `/evolution`

**Interfaces:**
- Consumes: `@casehubio/mosaic-evolution-workbench` (custom element `agency-evolution-workbench`), `@casehubio/mosaic-evolution-config` (types/sub-components), `@casehubio/pages-ui` DSL (`page`, `hostPanel`), `@casehubio/pages-runtime` (`registerPanel`, `loadSite`)
- Produces: `/evolution` route rendering the evolution workbench; `/` route unchanged (review dashboard)

- [ ] **Step 1: Add evolution package dependencies to package.json**

Add `@casehubio/mosaic-evolution-workbench` and `@casehubio/mosaic-evolution-config` to `devtown/app/src/main/webui/package.json` dependencies:

```json
"@casehubio/mosaic-evolution-config": "file:.casehub-packages/packages/evolution-config",
"@casehubio/mosaic-evolution-workbench": "file:.casehub-packages/packages/evolution-workbench",
```

Insert alphabetically among the existing `@casehubio/mosaic-*` entries.

- [ ] **Step 2: Run npm install to link packages**

Run:
```bash
cd devtown/app/src/main/webui && npm install
```
Expected: clean install, no errors. The packages are already unpacked at `.casehub-packages/packages/`.

- [ ] **Step 3: Create evolution view file**

Create `devtown/app/src/main/webui/src/views/evolution.ts`:

```typescript
import { page, hostPanel } from "@casehubio/pages-ui";

export const evolutionView = page("DevTown Evolution",
  hostPanel("evolution-workbench", { endpoint: "/api/devtown/evolution" }),
);
```

This creates a full-page experience (not tabs) hosting the evolution workbench with the backend endpoint.

- [ ] **Step 4: Add routing and evolution panel registration to index.ts**

Modify `devtown/app/src/main/webui/src/index.ts`:

1. Add import for the evolution workbench custom element and view at the top:

```typescript
import "@casehubio/mosaic-evolution-workbench";
import "@casehubio/mosaic-evolution-config";
import { evolutionView } from "./views/evolution";
```

2. Add panel registration after the existing `registerPanel` calls:

```typescript
registerPanel("evolution-workbench", "agency-evolution-workbench");
```

3. In the `start()` function, add path-based routing. Replace the section that builds `app` and calls `loadSite` with:

```typescript
const path = window.location.pathname;
const isEvolution = path.startsWith("/evolution");

const app = isEvolution
  ? evolutionView
  : page("DevTown",
      tabs(
        ["PR Cases", hostPanel("pr-cases-workbench", { endpoint: "/api/devtown/governance" })],
        ["Reviewers", reviewersView],
        ["Contributors", hostPanel("contributor-workbench", { endpoint: "/api/devtown/reviews" })],
        ["Workers", hostPanel("session-workbench", { endpoint: "/api/sessions" })],
        ["System", hostPanel("system-workbench", { endpoint: "/api/devtown/governance" })],
        ["Definitions", definitionsView],
      ),
      { settings: { mode: prefersDark ? "dark" : "light" }, datasets: createDatasets(prefs) },
    );

const container = document.getElementById("app");
if (container) {
  const siteOptions: SiteOptions = {
    permissions: permissionsFrom(user),
    fetch: authFetch,
  };
  const site = await loadSite(container, app, siteOptions);
  site.setTheme(prefersDark ? "dark" : "light");

  window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", (e) => {
    site.setTheme(e.matches ? "dark" : "light");
  });
}
```

Import `SiteOptions` type at the top alongside `loadSite`:

```typescript
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import type { SiteOptions } from "@casehubio/pages-runtime";
```

- [ ] **Step 5: Build and verify**

Run:
```bash
cd devtown/app/src/main/webui && npm run build
```
Expected: clean build, `dist/app.js` produced without errors.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/webui/package.json app/src/main/webui/src/views/evolution.ts app/src/main/webui/src/index.ts
git commit -m "feat(#1225): wire evolution workbench as standalone page at /evolution

Add evolution-workbench and evolution-config package dependencies.
Create evolution.ts view with full-page hostPanel mounting.
Add client-side path routing in index.ts — /evolution renders
the evolution workbench, all other paths render the review dashboard.

Refs #1225"
```

---

### Task 2: Fix DevtownEvolutionBootstrap startup observer

The issue reports that the startup observer doesn't create the evolution case. The current implementation uses `@Observes StartupEvent` which fires during Quarkus startup. `DevtownEvolutionCaseHub` extends `YamlCaseHub` and calls `super("casehub/devtown/evolution.yaml")`. The `startCase()` method on `CaseHub` delegates to `CaseHubRuntime.startCase(getDefinition())`. Possible failure modes: (a) `CaseHubRuntime` not injected yet at startup time, (b) YAML parsing fails silently, (c) `getDefinition()` throws but the exception is swallowed.

**Files:**
- Modify: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrap.java` — add error handling, consider `@Observes StartupEvent` priority
- Modify: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrapTest.java` — add failure-mode tests
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrapTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository.findByNamespaceAndName()`, `DevtownEvolutionCaseHub.startCase()`
- Produces: A reliably-created evolution case instance at startup

- [ ] **Step 1: Write a test for the failure-case scenario**

Add a test to `DevtownEvolutionBootstrapTest.java` that verifies the bootstrap handles `startCase()` failure gracefully:

```java
@Test
void logs_error_when_case_creation_fails() {
  when(repository.findByNamespaceAndName("devtown", "evolution", "default"))
      .thenReturn(List.of());
  doThrow(new RuntimeException("YAML parse error"))
      .when(caseHub).startCase();

  // Should not propagate — startup must not fail
  bootstrap.onStartup(Mockito.mock(StartupEvent.class));

  verify(caseHub).startCase();
}
```

Add import: `import static org.mockito.Mockito.doThrow;`

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionBootstrapTest -f devtown/pom.xml
```
Expected: FAIL — `logs_error_when_case_creation_fails` throws `RuntimeException` because the bootstrap doesn't catch it.

- [ ] **Step 3: Fix the bootstrap to handle exceptions**

Modify `DevtownEvolutionBootstrap.java` to wrap `startCase()` in a try-catch with logging:

```java
package io.casehub.devtown.app.evolution;

import io.casehub.engine.common.spi.CaseInstanceRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import io.quarkus.runtime.StartupEvent;
import org.jboss.logging.Logger;

@ApplicationScoped
public class DevtownEvolutionBootstrap {

  private static final Logger LOG = Logger.getLogger(DevtownEvolutionBootstrap.class);

  static final String NAMESPACE = "devtown";
  static final String NAME = "evolution";
  static final String TENANCY_ID = "default";

  @Inject CaseInstanceRepository caseInstanceRepository;
  @Inject DevtownEvolutionCaseHub caseHub;

  void onStartup(@Observes StartupEvent event) {
    var existing = caseInstanceRepository.findByNamespaceAndName(NAMESPACE, NAME, TENANCY_ID);
    if (existing.isEmpty()) {
      try {
        caseHub.startCase();
        LOG.info("Created evolution case for devtown");
      } catch (Exception e) {
        LOG.errorf(e, "Failed to create evolution case — evolution features will be unavailable until restart");
      }
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionBootstrapTest -f devtown/pom.xml
```
Expected: all 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrap.java app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrapTest.java
git commit -m "fix(#1225): handle startup failure in DevtownEvolutionBootstrap

Wrap caseHub.startCase() in try-catch so a failed evolution case
creation doesn't crash Quarkus startup. Log the error and continue —
evolution features will be unavailable but the review dashboard works.

Refs #1225"
```

---

## Batch 2: Cross-App Integration

After this batch, conductor-driven PRs show an origin indicator in the review detail, and evolution streams link to review detail pages.

### Task 3: Add conductor-origin indicator to PR detail

When a PR review case was initiated by the evolution conductor (i.e., an improvement stream drove the configuration change that resulted in the PR), the review detail panel should show a small indicator linking back to the evolution stream. This requires: (a) a backend endpoint to check if a case has conductor origin, and (b) a frontend badge in pr-detail.ts.

**Files:**
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/ConductorOriginView.java` — view record
- Modify: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionResource.java` — add conductor-origin lookup endpoint
- Modify: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionApi.java` — add conductor-origin lookup method
- Create: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiConductorOriginTest.java` — test
- Modify: `devtown/app/src/main/webui/src/components/pr-detail.ts` — fetch and render conductor origin badge
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiConductorOriginTest.java`

**Interfaces:**
- Consumes: `EngineEvolutionApi.getStreamProgress()` to check if any active stream targets a case ID, `ReviewDetailData.caseId` from the PR detail
- Produces: `GET /api/devtown/evolution/conductor-origin/{caseId}` returning `ConductorOriginView` (nullable), conductor-origin badge in the PR detail component

- [ ] **Step 1: Create ConductorOriginView record**

Create `devtown/app/src/main/java/io/casehub/devtown/app/evolution/ConductorOriginView.java`:

```java
package io.casehub.devtown.app.evolution;

import jakarta.annotation.Nullable;

public record ConductorOriginView(
    String streamCategory,
    @Nullable String streamTarget,
    String currentStage) {}
```

- [ ] **Step 2: Write failing test for conductor-origin lookup**

Create `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiConductorOriginTest.java`:

```java
package io.casehub.devtown.app.evolution;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.when;

import io.casehub.api.spi.improvement.EngineEvolutionApi;
import io.casehub.api.view.EvolutionStateSnapshot.ImprovementStreamView;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

class DevtownEvolutionApiConductorOriginTest {

  private EngineEvolutionApi engineApi;
  private DevtownEvolutionCaseResolver caseResolver;
  private DevtownEvolutionEnricher enricher;
  private DevtownEvolutionApi api;
  private final String tenancyId = "default";
  private final UUID evolutionCaseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    engineApi = Mockito.mock(EngineEvolutionApi.class);
    caseResolver = Mockito.mock(DevtownEvolutionCaseResolver.class);
    enricher = Mockito.mock(DevtownEvolutionEnricher.class);
    api = new DevtownEvolutionApi();
    api.engineApi = engineApi;
    api.caseResolver = caseResolver;
    api.enricher = enricher;
    when(caseResolver.resolve(tenancyId)).thenReturn(evolutionCaseId);
  }

  @Test
  void returns_origin_when_stream_targets_case() {
    var targetCaseId = UUID.randomUUID();
    var stream = new ImprovementStreamView(
        UUID.randomUUID(), "reviewer-calibration", targetCaseId.toString(),
        "propose", List.of(), null, false, Instant.now());
    when(engineApi.getStreamProgress(evolutionCaseId, tenancyId))
        .thenReturn(List.of(stream));

    var origin = api.getConductorOrigin(tenancyId, targetCaseId.toString());

    assertNotNull(origin);
    assertEquals("reviewer-calibration", origin.streamCategory());
    assertEquals("propose", origin.currentStage());
  }

  @Test
  void returns_null_when_no_stream_targets_case() {
    var stream = new ImprovementStreamView(
        UUID.randomUUID(), "sla-tuning", "other-target",
        "analyze", List.of(), null, false, Instant.now());
    when(engineApi.getStreamProgress(evolutionCaseId, tenancyId))
        .thenReturn(List.of(stream));

    var origin = api.getConductorOrigin(tenancyId, "unrelated-case-id");

    assertNull(origin);
  }

  @Test
  void returns_null_when_no_streams_exist() {
    when(engineApi.getStreamProgress(evolutionCaseId, tenancyId))
        .thenReturn(List.of());

    var origin = api.getConductorOrigin(tenancyId, "any-case-id");

    assertNull(origin);
  }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionApiConductorOriginTest -f devtown/pom.xml
```
Expected: FAIL — `getConductorOrigin` method doesn't exist yet.

- [ ] **Step 4: Implement getConductorOrigin in DevtownEvolutionApi**

Add method to `DevtownEvolutionApi.java`:

```java
public @Nullable ConductorOriginView getConductorOrigin(String tenancyId, String targetCaseId) {
  UUID caseId = caseResolver.resolve(tenancyId);
  var streams = engineApi.getStreamProgress(caseId, tenancyId);
  return streams.stream()
      .filter(s -> targetCaseId.equals(s.target()))
      .findFirst()
      .map(s -> new ConductorOriginView(s.category(), s.target(), s.currentStage()))
      .orElse(null);
}
```

Add import: `import jakarta.annotation.Nullable;`

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionApiConductorOriginTest -f devtown/pom.xml
```
Expected: all 3 tests PASS.

- [ ] **Step 6: Add REST endpoint for conductor-origin**

Add to `DevtownEvolutionResource.java`:

```java
@GET
@Path("/conductor-origin/{caseId}")
public ConductorOriginView getConductorOrigin(
    @PathParam("caseId") String caseId,
    @QueryParam("tenancyId") String tenancyId) {
  return api.getConductorOrigin(tenancyId, caseId);
}
```

- [ ] **Step 7: Add conductor-origin indicator to PR detail frontend**

Modify `devtown/app/src/main/webui/src/components/pr-detail.ts`:

1. Add a state field for conductor origin:

```typescript
@state() private _conductorOrigin: { streamCategory: string; streamTarget: string | null; currentStage: string } | null = null;
```

2. In `_fetchDetail()`, after fetching the detail data successfully, fetch conductor origin:

```typescript
// After: this._data = await res.json();
this._fetchConductorOrigin(this.caseId);
```

3. Add the fetch method:

```typescript
private async _fetchConductorOrigin(caseId: string): Promise<void> {
  this._conductorOrigin = null;
  try {
    const res = await fetch(`/api/devtown/evolution/conductor-origin/${caseId}`);
    if (res.ok) {
      const data = await res.json();
      if (data) this._conductorOrigin = data;
    }
  } catch { /* conductor origin is optional context */ }
}
```

4. In `render()`, after the PR header and before the actions, add the conductor origin indicator:

```typescript
${this._conductorOrigin ? html`
  <div class="conductor-origin">
    <a href="/evolution" class="conductor-link">
      ⚡ Conductor: ${this._conductorOrigin.streamCategory} · ${this._conductorOrigin.currentStage}
    </a>
  </div>
` : nothing}
```

5. Add styles for the indicator in the `static override styles` array:

```css
.conductor-origin {
  margin: 4px 0 8px;
  font-size: 12px;
}
.conductor-link {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 3px 8px;
  background: var(--pages-primary-2, #eff6ff);
  color: var(--pages-primary-9, #1d4ed8);
  border-radius: 3px;
  text-decoration: none;
  font-weight: 500;
}
.conductor-link:hover {
  background: var(--pages-primary-3, #dbeafe);
}
```

- [ ] **Step 8: Build frontend**

Run:
```bash
cd devtown/app/src/main/webui && npm run build
```
Expected: clean build.

- [ ] **Step 9: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/ConductorOriginView.java \
  app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionApi.java \
  app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionResource.java \
  app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiConductorOriginTest.java \
  app/src/main/webui/src/components/pr-detail.ts
git commit -m "feat(#1225): conductor-origin indicator on PR detail

Add GET /api/devtown/evolution/conductor-origin/{caseId} endpoint
that checks if any active improvement stream targets the given case.
Show a small badge in the PR detail linking back to /evolution when
a PR was conductor-driven.

Refs #1225"
```

---

### Task 4: Cross-app deep linking from evolution to review

Evolution streams display a `target` field that identifies what the improvement targets (e.g., a case ID, a reviewer ID). When the target is a case ID corresponding to a PR review case, clicking it should navigate to the review detail for that case. The evolution workbench renders targets as text (design decision D6 — no blocks-ui changes). The enricher can format these as navigable paths.

**Files:**
- Modify: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricher.java` — enrich stream targets with review deep links
- Modify: `devtown/app/src/main/java/io/casehub/devtown/app/evolution/DevtownImprovementStreamView.java` — add deepLink field
- Modify: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricherTest.java` — add deep link tests
- Test: `devtown/app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricherTest.java`

**Interfaces:**
- Consumes: `ImprovementStreamView.target()`, `ImprovementStreamView.category()` from engine API
- Produces: `DevtownImprovementStreamView.deepLink()` — nullable URL path for client-side navigation (e.g., `/?case=<uuid>`)

- [ ] **Step 1: Write failing test for deep link enrichment**

Add tests to `DevtownEvolutionEnricherTest.java`:

```java
@Test
void enriches_reviewer_calibration_stream_with_reviewer_deep_link() {
  var stream = createStream("reviewer-calibration", "reviewer:agent-security");
  var state = createState(List.of(stream));

  var result = enricher.enrich(state);

  var enriched = result.enrichedStreams().get(0);
  assertNotNull(enriched.deepLink());
  assertEquals("/?reviewers=agent-security", enriched.deepLink());
}

@Test
void enriches_queue_optimization_stream_without_deep_link() {
  var stream = createStream("queue-optimization", null);
  var state = createState(List.of(stream));

  var result = enricher.enrich(state);

  var enriched = result.enrichedStreams().get(0);
  assertNull(enriched.deepLink());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionEnricherTest -f devtown/pom.xml
```
Expected: FAIL — `deepLink()` field doesn't exist on `DevtownImprovementStreamView`.

- [ ] **Step 3: Add deepLink field to DevtownImprovementStreamView**

Modify `DevtownImprovementStreamView.java`:

```java
package io.casehub.devtown.app.evolution;

import io.casehub.api.view.EvolutionStateSnapshot.ImprovementStreamView;
import jakarta.annotation.Nullable;

public record DevtownImprovementStreamView(
    ImprovementStreamView stream,
    @Nullable String enrichedTarget,
    @Nullable String deepLink) {}
```

- [ ] **Step 4: Update enricher to produce deep links**

Modify `DevtownEvolutionEnricher.java` — update `enrichStream` to compute a deep link based on category and target:

```java
private DevtownImprovementStreamView enrichStream(ImprovementStreamView stream) {
  String target = stream.target();
  String enrichedTarget = resolveTarget(stream.category(), target);
  String deepLink = resolveDeepLink(stream.category(), target);
  return new DevtownImprovementStreamView(stream, enrichedTarget, deepLink);
}

private String resolveDeepLink(String category, String target) {
  if (target == null) return null;
  return switch (category) {
    case "reviewer-calibration" -> {
      // target format: "reviewer:<actorId>"
      if (target.startsWith("reviewer:")) {
        yield "/?reviewers=" + target.substring("reviewer:".length());
      }
      yield null;
    }
    case "routing-adjustment" -> {
      if (target.startsWith("reviewer:")) {
        yield "/?reviewers=" + target.substring("reviewer:".length());
      }
      yield null;
    }
    default -> null;
  };
}
```

- [ ] **Step 5: Update any callers of DevtownImprovementStreamView constructor**

Search for existing uses of the 2-arg constructor and update to 3-arg. The enricher's `enrichStream` already uses the new signature. Check if any test helpers create `DevtownImprovementStreamView` directly and update them.

- [ ] **Step 6: Run tests to verify they pass**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionEnricherTest -f devtown/pom.xml
```
Expected: all tests PASS.

- [ ] **Step 7: Run full evolution test suite**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest="io.casehub.devtown.app.evolution.*" -f devtown/pom.xml
```
Expected: all 14+ tests PASS (existing + new).

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/DevtownImprovementStreamView.java \
  app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricher.java \
  app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricherTest.java
git commit -m "feat(#1225): cross-app deep links from evolution streams to review

Enrich DevtownImprovementStreamView with a deepLink field that maps
stream targets to review app URLs. reviewer-calibration and
routing-adjustment streams targeting a reviewer produce a link to
the reviewers view.

Refs #1225"
```

---

## References

- [2026-10-07-evolution-conductor-ui-in-devtown-design.md] — original design spec (decisions D1–D7)
- [decisions.md] — design decisions, especially D4 (originally tab, now standalone) and D6 (deep links via enrichment)
- [devtown/app/src/main/webui/src/index.ts] — current entry point with 6-tab dashboard
- [devtown/app/src/main/java/io/casehub/devtown/app/evolution/] — existing backend code (14 classes)
- [GitHub #1225] — restructure evolution as standalone app
- [GitHub #1180] — original evolution conductor UI implementation
- [devtown/app/src/main/webui/.casehub-packages/packages/evolution-workbench/] — pre-built UI component
