# The Schema Already Knew

The YAML schema for agent model configuration had `baseUrl` on `OpenAiModel` since the schema was written. The generated Java class had `getBaseUrl()` and `setBaseUrl()`. The LangChain4j builder accepted `baseUrl(String)`. Every layer knew about it — except the one layer that matters: `OpenAiChatModelProvider`, which silently dropped the field on the floor.

Same story for `organizationId`, `frequencyPenalty`, `presencePenalty` on OpenAI. Same for `baseUrl` and `version` on Anthropic. Same for `baseUrl` on Mistral. Seven fields across three providers, all declared in schema, all present in generated model classes, all accepted by LangChain4j — and all silently lost when `AgentConverter` built the provider.

The fix is mechanical. Add the field to the provider builder, wire it through reflection in `get()`, wire it in `AgentConverter.toXxxProvider()`. The only interesting part was the scope decision: the issue asked for OpenAI's `baseUrl` only, but once you see one gap you see them all. We fixed all three providers in one pass.

Two other small SPI additions landed on the same branch. `CaseDefinitionRegistry.findByIdentity()` returns `Optional<CaseMetaModel>` instead of the existing `getCaseMetaModel()` which throws on not-found — a clean existence query for the ops drift-detection bridge. And three query methods on `CaseInstanceRepository` (`findByStatus`, `findAll`, `findByNamespaceAndName`) that application tiers need for operational tooling. Both follow the SPI evolution protocol: `default` methods with safe no-op returns, so existing implementations compile without changes.

The provider gap is the one that nags. Silent data loss is the worst category of bug — everything works, the tests pass, the YAML validates, and the field you carefully configured just doesn't reach the LLM call. If `casehub-life` had deployed with an `openclaw` baseUrl in YAML, it would have silently called OpenAI's production endpoint instead. No error, wrong backend, production data to the wrong place.
