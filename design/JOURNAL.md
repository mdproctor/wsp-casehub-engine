# Design Journal — issue-478-case-retriever-routing-bridge

## §1 CBR Retrieval Bridge Design (2026-07-06)

**Decision:** Push-based context enrichment over pull-based strategy injection.

Trust scoring uses pull (`TrustScoreSource.getScore(workerId, capabilityName)`) because
the strategy already has both lookup keys. CBR retrieval requires `CaseDefinition.CbrConfig`
+ `CaseContext` for domain-specific feature extraction — data the engine has at the call
site but a strategy would need to re-resolve via repository lookups. The information
asymmetry makes push the correct pattern for CBR.

**Decision:** Sealed `FeatureExtractor` over open `ExpressionEvaluator` subtyping.

`FeatureExtractor` is sealed (permits `JqFeatureExtractor`, `LambdaFeatureExtractor`).
`CbrRetrievalService` dispatches via pattern matching — the compiler enforces exhaustiveness.
Third-party extraction modes are not supported; two modes (declarative JQ, programmatic
lambda) cover YAML and Java DSL respectively. Distinct from `ExpressionEvaluator` which is
open and dispatched by `ExpressionEngine` registry.

**Decision:** Engine-owned `RetrievedExperience` over neocortex `ScoredCbrCase`.

Routing context types in `engine-api` cannot depend on `casehub-neocortex-memory-api` (tier
violation). `RetrievedExperience` and `ExperiencePlanStep` mirror `ScoredCbrCase<PlanCbrCase>`
and `PlanTrace` structurally. The runtime maps between them in `CbrRetrievalService`.

**Decision:** `CbrRetrievalService` injects blocking `CbrCaseMemoryStore` directly.

The reactive bridge (`BlockingToReactiveCbrBridge`) in neocortex resolves its delegate to the
no-op `@DefaultBean` instead of the `@Alternative` in-memory store — a Quarkus ARC build-time
resolution issue (GE-20260706-abaddc). Bypassing the bridge and using `runSubscriptionOn()` for
thread safety is the correct workaround.

**Open:** Integration tests `@Disabled` pending #675 — `getEnabledAlternatives()` in
`QuarkusTestProfile` replaces all `selected-alternatives` globally, breaking other tests.
Needs a `CbrMemoryProfile` that re-declares all required alternatives.
