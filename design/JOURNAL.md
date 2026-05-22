# Design Journal — issue-320-consume-platform-expression

### 2026-05-22 · §Dependencies and SPI

`JQEvaluator`, `ValidationResult`, `SecretManager`, `ConfigManager`, and their
exception types were removed from `casehub-engine-common` and replaced with
equivalents from `casehub-platform-expression` and `casehub-platform-api`.
The engine-common interim home (introduced in engine#314) was always temporary —
a domain-agnostic JQ evaluator belongs in the Foundation tier so it can be
consumed by platform peers without pulling in Orchestration-tier dependencies.
Engine implementations (`QuarkusConfigManager`, `ConfigSecretManager`) remain in
engine/runtime as `@ApplicationScoped` beans that displace the platform's
`@DefaultBean` mocks automatically. `ConfigContext` stays in engine-common as an
engine-internal holder interface.
