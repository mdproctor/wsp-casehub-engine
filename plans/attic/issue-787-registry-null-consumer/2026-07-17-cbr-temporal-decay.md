# CbrConfig temporalDecayHalfLifeDays Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #733 — feat: add temporalDecayHalfLifeDays to CbrConfig
**Issue group:** #733

**Goal:** Let case definitions configure how quickly older CBR cases lose relevance during retrieval, via a `temporalDecayHalfLifeDays` field on `CbrConfig`.

**Architecture:** Add a nullable `Integer` to the `CbrConfig` record. The YAML schema, builder, and mapper all gain the field. `CbrRetrievalService` converts it to `TemporalDecay.HalfLife(Duration.ofDays(n))` on the `CbrQuery` at retrieval time. Null means no decay (backward compatible).

**Tech Stack:** Java 21 records, jsonschema2pojo codegen, JUnit 5, AssertJ

## Global Constraints

- Pre-release: breaking changes are free
- No new dependencies — `TemporalDecay` is already on the runtime classpath via neocortex
- Null = no decay (backward compatible default)
- IntelliJ MCP required for all Java edits

---

### Task 1: CbrConfig field + YAML schema + mapper

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` (Cbr definition, ~line 306)
- Modify: `api/src/main/java/io/casehub/api/model/cbr/CbrConfig.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (~line 564)
- Test: `api/src/test/java/io/casehub/api/model/cbr/CbrConfigBuilderTest.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperCbrTest.java`

**Interfaces:**
- Produces: `CbrConfig.temporalDecayHalfLifeDays()` returning `Integer` (nullable), `CbrConfig.Builder.temporalDecayHalfLifeDays(Integer)` returning `Builder`

- [ ] **Step 1: Add `temporalDecayHalfLifeDays` to the YAML schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, add to the `Cbr` definition after the `timing` property:

```yaml
      temporalDecayHalfLifeDays:
        type: integer
        minimum: 1
        description: >
          Half-life in days for temporal decay during CBR retrieval. Older cases
          lose relevance exponentially: similarity *= 0.5^(age / halfLife).
          Null means no temporal decay (default).
```

- [ ] **Step 2: Rebuild schema module to regenerate `Cbr` class**

Run: `mvn install -DskipTests -q -pl schema`

Verify: `target/generated-sources/jsonschema2pojo/io/casehub/model/Cbr.java` contains `getTemporalDecayHalfLifeDays()`.

- [ ] **Step 3: Write failing tests for CbrConfig**

Add to `CbrConfigBuilderTest.java`:

```java
@Test
void temporalDecayHalfLifeDays_defaults_to_null() {
  var config = CbrConfig.builder().feature("f1", ".x").build();
  assertNull(config.temporalDecayHalfLifeDays());
}

@Test
void temporalDecayHalfLifeDays_set_via_builder() {
  var config = CbrConfig.builder().feature("f1", ".x").temporalDecayHalfLifeDays(30).build();
  assertEquals(30, config.temporalDecayHalfLifeDays());
}

@Test
void temporalDecayHalfLifeDays_zero_rejected() {
  assertThrows(
      IllegalArgumentException.class,
      () -> CbrConfig.builder().feature("f1", ".x").temporalDecayHalfLifeDays(0).build());
}

@Test
void temporalDecayHalfLifeDays_negative_rejected() {
  assertThrows(
      IllegalArgumentException.class,
      () -> CbrConfig.builder().feature("f1", ".x").temporalDecayHalfLifeDays(-5).build());
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CbrConfigBuilderTest -q`

Expected: compilation failure — `temporalDecayHalfLifeDays` does not exist on CbrConfig yet.

- [ ] **Step 5: Add field to CbrConfig record and builder**

In `CbrConfig.java`:

Record component — add `Integer temporalDecayHalfLifeDays` as the last parameter (after `cbrType`).

Compact constructor — add validation:
```java
if (temporalDecayHalfLifeDays != null && temporalDecayHalfLifeDays < 1) {
  throw new IllegalArgumentException(
      "temporalDecayHalfLifeDays must be >= 1 when provided, got: " + temporalDecayHalfLifeDays);
}
```

Builder — add field and method:
```java
private Integer temporalDecayHalfLifeDays;

public Builder temporalDecayHalfLifeDays(final Integer temporalDecayHalfLifeDays) {
  this.temporalDecayHalfLifeDays = temporalDecayHalfLifeDays;
  return this;
}
```

Builder.build() — pass `temporalDecayHalfLifeDays` as the last argument to the `CbrConfig` constructor.

- [ ] **Step 6: Run CbrConfig tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CbrConfigBuilderTest -q`

Expected: all pass.

- [ ] **Step 7: Write failing YAML mapper test**

Add to `CaseDefinitionYamlMapperCbrTest.java`:

```java
@Test
void cbr_temporalDecayHalfLifeDays_parsed() throws IOException {
  String yaml =
      """
      dsl: "0.1.0"
      namespace: test
      name: test-case
      version: "1.0.0"
      spec:
        cbr:
          features:
            f1: ".x"
          temporalDecayHalfLifeDays: 30
      """;
  InputStream is = new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8));
  CaseDefinition def = CaseDefinitionYamlMapper.load(is);
  assertThat(def.getCbrConfig()).isNotNull();
  assertThat(def.getCbrConfig().temporalDecayHalfLifeDays()).isEqualTo(30);
}

@Test
void cbr_without_temporalDecay_defaults_to_null() throws IOException {
  String yaml =
      """
      dsl: "0.1.0"
      namespace: test
      name: test-case
      version: "1.0.0"
      spec:
        cbr:
          features:
            f1: ".x"
      """;
  InputStream is = new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8));
  CaseDefinition def = CaseDefinitionYamlMapper.load(is);
  assertThat(def.getCbrConfig()).isNotNull();
  assertThat(def.getCbrConfig().temporalDecayHalfLifeDays()).isNull();
}
```

- [ ] **Step 8: Run mapper tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperCbrTest -q`

Expected: `cbr_temporalDecayHalfLifeDays_parsed` fails — mapper does not wire the field yet.

- [ ] **Step 9: Wire temporalDecayHalfLifeDays in CaseDefinitionYamlMapper**

In `CaseDefinitionYamlMapper.java`, after the `cbrType` block (~line 567), add:

```java
if (cbr.getTemporalDecayHalfLifeDays() != null) {
  cbrBuilder.temporalDecayHalfLifeDays(cbr.getTemporalDecayHalfLifeDays());
}
```

- [ ] **Step 10: Run all CBR-related tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CbrConfigBuilderTest,CaseDefinitionYamlMapperCbrTest" -q`

Expected: all pass.

- [ ] **Step 11: Commit**

```bash
git add schema/src/main/resources/schema/CaseDefinition.yaml \
  api/src/main/java/io/casehub/api/model/cbr/CbrConfig.java \
  api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java \
  api/src/test/java/io/casehub/api/model/cbr/CbrConfigBuilderTest.java \
  api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperCbrTest.java
git commit -m "feat(cbr): add temporalDecayHalfLifeDays to CbrConfig

Nullable Integer field (default null = no decay). YAML schema, builder,
and mapper all wired. Zero and negative values rejected.

Refs #733"
```

---

### Task 2: Wire CbrRetrievalService to pass temporal decay to CbrQuery

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java` (~line 150, query construction)
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java`

**Interfaces:**
- Consumes: `CbrConfig.temporalDecayHalfLifeDays()` (from Task 1)
- Consumes: `CbrQuery.withTemporalDecay(TemporalDecay)` (existing neocortex API)
- Consumes: `TemporalDecay.HalfLife(Duration)` (existing neocortex type)

- [ ] **Step 1: Install api so runtime sees the new field**

Run: `mvn install -DskipTests -q -pl schema,api`

- [ ] **Step 2: Write failing test**

Add to `CbrRetrievalServiceTest.java`:

```java
@Test
void temporalDecay_set_on_query_when_configured() {
  CbrConfig config =
      CbrConfig.builder()
          .featureExtractor(ctx -> Map.of("f1", "v1"))
          .domain("test")
          .temporalDecayHalfLifeDays(30)
          .build();
  CaseDefinition def = buildDefinition(config);
  cbrStore.setResult(List.of());
  service.retrieve(def, buildInstance()).await().indefinitely();

  CbrQuery query = cbrStore.lastQuery();
  assertNotNull(query.temporalDecay());
  assertInstanceOf(TemporalDecay.HalfLife.class, query.temporalDecay());
  TemporalDecay.HalfLife halfLife = (TemporalDecay.HalfLife) query.temporalDecay();
  assertEquals(Duration.ofDays(30), halfLife.halfLife());
}

@Test
void temporalDecay_null_when_not_configured() {
  CbrConfig config =
      CbrConfig.builder()
          .featureExtractor(ctx -> Map.of("f1", "v1"))
          .domain("test")
          .build();
  CaseDefinition def = buildDefinition(config);
  cbrStore.setResult(List.of());
  service.retrieve(def, buildInstance()).await().indefinitely();

  assertNull(cbrStore.lastQuery().temporalDecay());
}
```

Imports to add:
```java
import io.casehub.neocortex.memory.cbr.TemporalDecay;
import java.time.Duration;
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CbrRetrievalServiceTest#temporalDecay_set_on_query_when_configured -q`

Expected: FAIL — `temporalDecay()` returns null because CbrRetrievalService does not set it.

- [ ] **Step 4: Wire temporal decay in CbrRetrievalService**

In `CbrRetrievalService.java`, in the `retrieveInternal` method, after the query construction block (`.withVectorWeight(config.vectorWeight())`), add:

```java
if (config.temporalDecayHalfLifeDays() != null) {
  query = query.withTemporalDecay(
      new TemporalDecay.HalfLife(Duration.ofDays(config.temporalDecayHalfLifeDays())));
}
```

Import to add:
```java
import io.casehub.neocortex.memory.cbr.TemporalDecay;
import java.time.Duration;
```

Note: the existing code builds `query` as a final local and chains `with*()` methods. The temporal decay line needs the query to be reassignable — change `CbrQuery query =` to non-final (it's a local variable in a lambda, but it's not effectively final after this change — assign to a new variable instead):

```java
CbrQuery query =
    CbrQuery.of(...)
        .withMinSimilarity(config.minSimilarity())
        .withWeights(config.weights())
        .withVectorWeight(config.vectorWeight());

if (config.temporalDecayHalfLifeDays() != null) {
  query = query.withTemporalDecay(
      new TemporalDecay.HalfLife(Duration.ofDays(config.temporalDecayHalfLifeDays())));
}
```

- [ ] **Step 5: Run both temporal decay tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CbrRetrievalServiceTest#temporalDecay_set_on_query_when_configured+temporalDecay_null_when_not_configured" -q`

Expected: both pass.

- [ ] **Step 6: Run full CbrRetrievalServiceTest suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CbrRetrievalServiceTest -q`

Expected: all existing tests still pass (no regression).

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java \
  runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java
git commit -m "feat(cbr): wire temporalDecayHalfLifeDays through CbrRetrievalService

When CbrConfig.temporalDecayHalfLifeDays is non-null, CbrRetrievalService
converts to TemporalDecay.HalfLife(Duration.ofDays(n)) on the CbrQuery.
Null leaves temporalDecay unset (no decay, backward compatible).

Refs #733"
```

---

### Task 3: Update CLAUDE.md and verify full build

**Files:**
- Modify: `CLAUDE.md` (CbrConfig section)

- [ ] **Step 1: Update CLAUDE.md CbrConfig documentation**

In the `CbrConfig on CaseDefinition` section, add `temporalDecayHalfLifeDays` to the field list after `cbrType`:

```
`CbrConfig` on `CaseDefinition` — configures CBR retrieval per case type: ... `cbrType` (string, nullable ...), `temporalDecayHalfLifeDays` (Integer, nullable — half-life in days for temporal decay; null = no decay). YAML `cbr:` block ...
```

- [ ] **Step 2: Full build verification**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime -q`

Expected: clean build, all tests pass.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add temporalDecayHalfLifeDays to CLAUDE.md CbrConfig section

Refs #733"
```
