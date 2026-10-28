## D1: API shape for sub-case identity

**Choice:** Extend PlanItemView with `capabilityName` and `subCaseIds`
**Alternatives:**
- Dedicated `getSubCases` endpoint — clean separation but forces two API calls and client-side join
- Sealed hierarchy (`SubCasePlanItemView extends PlanItemView`) — type-safe but breaking API change, over-engineered for two fields
**Rationale:** Single API call from consumers. The nullable fields are a minor wart justified by not forcing a second call. The issue's own framing anticipated this approach.
**Trade-offs:** PlanItemView gains nullable fields only relevant to sub-case target type.
**Sources:** casehubio/engine#1260, casehubio/scaffold#74, `api/src/main/java/io/casehub/api/view/PlanItemView.java`
**Exploration:** quick
**Status:** captured

## D2: M-of-N representation

**Choice:** `List<UUID> subCaseIds` — empty when not yet spawned, one element for ungrouped, N elements for M-of-N
**Alternatives:**
- Single ID + group query — simpler field but requires separate query for grouped scenarios
- Flatten to one row per child — duplicates plan item rows, confuses consumers expecting 1:1
**Rationale:** Single field handles all cardinalities. Consumers iterate the list regardless of grouped/ungrouped.
**Trade-offs:** List field is slightly more complex than a single UUID for the common ungrouped case.
**Sources:** `common-core/src/main/java/io/casehub/engine/common/internal/model/SubCaseGroup.java`, `planning-core/src/main/java/io/casehub/engine/planning/subcase/SubCaseExecutionHandler.java`
**Exploration:** quick
**Depends on:** D1 (API shape)
**Status:** captured

## D3: Data source for child case IDs

**Choice:** Query SubCaseGroupRepository
**Alternatives:**
- Query EventLog — SUBCASE_STARTED events have the data but EventLog is an audit trail, not a query source
- Add column to PlanItemEntity — cleanest query path but schema change on the core entity
**Rationale:** SubCaseGroup already tracks parentCaseId → childCaseIds. Add a findByParentCaseId query. For ungrouped sub-cases, either register a synthetic group or store on PlanItemRecord.
**Trade-offs:** Ungrouped sub-cases currently don't create SubCaseGroup entries — need to either register them or use a fallback source.
**Sources:** `common-core/src/main/java/io/casehub/engine/common/spi/SubCaseGroupRepository.java`
**Exploration:** quick
**Depends on:** D1 (API shape)
**Status:** captured

## D4: Data source for capability name

**Choice:** Add `capabilityName` field to PlanItemSaveRequest/PlanItemRecord/PlanItemEntity
**Alternatives:**
- Look up from CaseDefinitionRegistry at query time — avoids schema change but adds per-item lookup
- Add to SubCaseGroup — keeps sub-case data together but requires SubCaseGroup schema change too
**Rationale:** The capability name is available from SubCase.name() when the plan item is created. Storing at write time gives zero-cost query. Consistent with how other plan item fields (executorName, description) are stored at creation.
**Trade-offs:** Requires adding a column to PlanItemEntity and fields to PlanItemSaveRequest/PlanItemRecord.
**Sources:** `api/src/main/java/io/casehub/api/model/SubCase.java`, `common-core/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java`
**Exploration:** quick
**Depends on:** D1 (API shape)
**Status:** captured
