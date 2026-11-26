# AgentTaskBindingResolver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/eidos#280 — Agent-task binding resolver for case YAML dispatch
**Issue group:** casehubio/eidos#280

**Goal:** Validate agent-task bindings at case definition registration time by checking that the AgentRegistry has agents matching each binding's capability/domain.

**Architecture:** A CDI startup observer in engine-runtime (not eidos-runtime — dependency analysis shows eidos can't depend on engine without creating a cycle). Runs at priority 2700, after YamlCaseHubDiscovery (2600). Iterates all CaseDefinitions, checks AgentTaskTarget bindings against AgentRegistry. Advisory warnings only.

**Tech Stack:** Java 21, Quarkus CDI, JUnit 5, Mockito, AssertJ

**Note on module placement:** The issue specified eidos-runtime, but CaseDefinitionRegistry lives in engine-common-core. Engine-runtime already depends on both engine-common-core and eidos-api (for AgentRegistry), making it the natural home. This follows the precedent of YamlCaseHubDiscovery and AgentWorkerFunctionHandler.

## Global Constraints

- Quarkus 3.32.2
- Test classes must be `*Test.java`, never `*IT.java`
- Inject by SPI interface, never concrete class
- `@ObservesAsync` is unreliable in `@QuarkusTest` — use direct method calls in tests

---

## Batch 1: AgentTaskBindingResolver + tests

### Task 1: AgentTaskBindingResolver — validation observer

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/runtime/bootstrap/AgentTaskBindingResolver.java`
- Test: `runtime/src/test/java/io/casehub/engine/runtime/bootstrap/AgentTaskBindingResolverTest.java`

**Interfaces:**
- Consumes: `CaseDefinitionRegistry.allDefinitions()` → `Collection<CaseDefinition>`, `CaseDefinition.getBindings()` → `List<Binding>`, `Binding.target()` → `BindingTarget` (check for `AgentTaskTarget`), `AgentTaskTarget.capability()` → `String`, `AgentTaskTarget.domain()` → `String`, `AgentRegistry.find(AgentQuery)` → `List<AgentMatch>`, `CurrentPrincipal.tenancyId()` → `String`
- Produces: Log output only — no API surface consumed by other tasks

- [ ] **Step 1: Write failing tests**

Create the test class with all five test cases. The resolver doesn't exist yet, so these won't compile.

```java
package io.casehub.engine.runtime.bootstrap;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;
import static org.mockito.Mockito.when;

import io.casehub.api.model.AgentTaskTarget;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.CaseDefinition;
import io.casehub.eidos.api.AgentCapability;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.platform.api.identity.CurrentPrincipal;
import java.util.Collection;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class AgentTaskBindingResolverTest {

  AgentTaskBindingResolver resolver;
  CaseDefinitionRegistry caseDefinitionRegistry;
  AgentRegistry agentRegistry;
  CurrentPrincipal principal;

  @BeforeEach
  void setUp() {
    caseDefinitionRegistry = mock(CaseDefinitionRegistry.class);
    agentRegistry = mock(AgentRegistry.class);
    principal = stubPrincipal("test-tenant");
    resolver = new AgentTaskBindingResolver();
    resolver.caseDefinitionRegistry = caseDefinitionRegistry;
    resolver.agentRegistry = agentRegistry;
    resolver.currentPrincipal = principal;
  }

  @Test
  void validate_bindingResolvesToAgent_noWarning() {
    AgentTaskTarget target = AgentTaskTarget.builder()
        .capability("text-classification").build();
    Binding binding = Binding.builder()
        .name("classify").agentTask(target).on("entry").build();
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("helpdesk").version("1.0")
        .bindings(List.of(binding)).build();
    when(caseDefinitionRegistry.allDefinitions()).thenReturn(List.of(def));
    when(agentRegistry.find(AgentQuery.byCapabilityAndDomain(
        "text-classification", null, "test-tenant")))
        .thenReturn(List.of(new AgentMatch(
            minimalAgent("classifier-agent"), null)));

    var result = resolver.validate();

    assertThat(result.resolved()).isEqualTo(1);
    assertThat(result.unresolved()).isEqualTo(0);
  }

  @Test
  void validate_bindingUnresolvable_countsAsUnresolved() {
    AgentTaskTarget target = AgentTaskTarget.builder()
        .capability("nonexistent-cap").build();
    Binding binding = Binding.builder()
        .name("missing-agent").agentTask(target).on("entry").build();
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("helpdesk").version("1.0")
        .bindings(List.of(binding)).build();
    when(caseDefinitionRegistry.allDefinitions()).thenReturn(List.of(def));
    when(agentRegistry.find(any())).thenReturn(List.of());

    var result = resolver.validate();

    assertThat(result.resolved()).isEqualTo(0);
    assertThat(result.unresolved()).isEqualTo(1);
  }

  @Test
  void validate_nonAgentTaskBinding_skipped() {
    CapabilityTarget capTarget = CapabilityTarget.builder()
        .capabilityName("do-work").build();
    Binding binding = Binding.builder()
        .name("do-stuff").capability(capTarget).on("entry").build();
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("basic").version("1.0")
        .bindings(List.of(binding)).build();
    when(caseDefinitionRegistry.allDefinitions()).thenReturn(List.of(def));

    var result = resolver.validate();

    assertThat(result.resolved()).isEqualTo(0);
    assertThat(result.unresolved()).isEqualTo(0);
    verifyNoInteractions(agentRegistry);
  }

  @Test
  void validate_multipleBindingsAcrossDefinitions() {
    AgentTaskTarget t1 = AgentTaskTarget.builder()
        .capability("classify").domain("legal").build();
    AgentTaskTarget t2 = AgentTaskTarget.builder()
        .capability("summarize").build();
    Binding b1 = Binding.builder()
        .name("b1").agentTask(t1).on("entry").build();
    Binding b2 = Binding.builder()
        .name("b2").agentTask(t2).on("entry").build();
    CaseDefinition def1 = CaseDefinition.builder()
        .namespace("test").name("case-a").version("1.0")
        .bindings(List.of(b1)).build();
    CaseDefinition def2 = CaseDefinition.builder()
        .namespace("test").name("case-b").version("1.0")
        .bindings(List.of(b2)).build();
    when(caseDefinitionRegistry.allDefinitions()).thenReturn(List.of(def1, def2));
    when(agentRegistry.find(AgentQuery.byCapabilityAndDomain(
        "classify", "legal", "test-tenant")))
        .thenReturn(List.of(new AgentMatch(minimalAgent("legal-agent"), null)));
    when(agentRegistry.find(AgentQuery.byCapabilityAndDomain(
        "summarize", null, "test-tenant")))
        .thenReturn(List.of());

    var result = resolver.validate();

    assertThat(result.resolved()).isEqualTo(1);
    assertThat(result.unresolved()).isEqualTo(1);
  }

  @Test
  void validate_emptyDefinitions_noOp() {
    when(caseDefinitionRegistry.allDefinitions()).thenReturn(List.of());

    var result = resolver.validate();

    assertThat(result.resolved()).isEqualTo(0);
    assertThat(result.unresolved()).isEqualTo(0);
    verifyNoInteractions(agentRegistry);
  }

  private static AgentDescriptor minimalAgent(String agentId) {
    return AgentDescriptor.builder()
        .agentId(agentId).name(agentId).slot("default")
        .tenancyId("test-tenant").build();
  }

  private static CurrentPrincipal stubPrincipal(String tenancyId) {
    return new CurrentPrincipal() {
      @Override public String actorId() { return "test-user"; }
      @Override public Set<String> groups() { return Set.of(); }
      @Override public String tenancyId() { return tenancyId; }
      @Override public boolean isCrossTenantAdmin() { return false; }
    };
  }
}
```

- [ ] **Step 2: Run tests to verify they fail (compilation error)**

Run: `mvn test-compile -pl runtime -q 2>&1 | tail -5`
Expected: compilation error — `AgentTaskBindingResolver` class not found

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.engine.runtime.bootstrap;

import io.casehub.api.model.AgentTaskTarget;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.runtime.StartupEvent;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class AgentTaskBindingResolver {

  private static final Logger LOG = Logger.getLogger(AgentTaskBindingResolver.class);

  @Inject CaseDefinitionRegistry caseDefinitionRegistry;
  @Inject AgentRegistry agentRegistry;
  @Inject CurrentPrincipal currentPrincipal;

  void onStart(@Observes @Priority(2700) StartupEvent event) {
    validate();
  }

  ValidationResult validate() {
    String tenancyId = currentPrincipal.tenancyId();
    int resolved = 0;
    int unresolved = 0;

    for (CaseDefinition def : caseDefinitionRegistry.allDefinitions()) {
      if (def.getBindings() == null) {
        continue;
      }
      for (Binding binding : def.getBindings()) {
        if (!(binding.target() instanceof AgentTaskTarget at)) {
          continue;
        }
        var matches = agentRegistry.find(
            AgentQuery.byCapabilityAndDomain(
                at.capability(), at.domain(), tenancyId));
        if (matches.isEmpty()) {
          LOG.warnf(
              "No agent found for agent-task binding '%s' in case '%s'"
                  + " (capability=%s, domain=%s)",
              binding.getName(), def.getName(),
              at.capability(), at.domain());
          unresolved++;
        } else {
          resolved++;
        }
      }
    }

    if (resolved + unresolved > 0) {
      LOG.infof(
          "AgentTaskBindingResolver: validated %d agent-task binding(s)"
              + " across case definitions — %d resolved, %d unresolved",
          resolved + unresolved, resolved, unresolved);
    }

    return new ValidationResult(resolved, unresolved);
  }

  record ValidationResult(int resolved, int unresolved) {}
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest='AgentTaskBindingResolverTest' -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5`
Expected: all 5 tests PASS

Note: If the runtime module has the pre-existing `SignalDedupExtendedTest` discovery failure, run with the `-Dtest` filter above or verify compilation + test correctness via `mvn test-compile -pl runtime -q` and inspection.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/bootstrap/AgentTaskBindingResolver.java
git add runtime/src/test/java/io/casehub/engine/runtime/bootstrap/AgentTaskBindingResolverTest.java
git commit -m "feat(#280): add AgentTaskBindingResolver — validate agent-task bindings at startup

Startup observer at priority 2700 that checks each AgentTaskTarget
binding against the AgentRegistry. Logs warnings for bindings with
no matching agent. Advisory only — agents may register dynamically.

Refs casehubio/eidos#280

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-11-26-agent-task-binding-resolver-design.md] — design spec
- [engine/common-core/.../CaseDefinitionRegistry.java:125] — `allDefinitions()` API
- [engine/api/.../Binding.java:35] — `target()` returns `BindingTarget`
- [engine/api/.../AgentTaskTarget.java] — capability/domain fields
- [eidos/api/AgentRegistry.java:22] — `find(AgentQuery)` API
- [eidos/api/AgentQuery.java:32] — `byCapabilityAndDomain()` factory
- [eidos/runtime/resolver/EidosAgentResolverTest.java] — test patterns (mocks, stubPrincipal)
- [engine/runtime/.../YamlCaseHubDiscovery.java] — startup observer precedent at priority 2600
- [GitHub casehubio/eidos#280] — focal issue
