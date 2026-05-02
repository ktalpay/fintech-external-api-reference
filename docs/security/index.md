# Security

This section collects security guidance for external API access.

Core themes:

- resolve company scope from trusted API key metadata,
- store only token hashes,
- deny invalid, expired, revoked, or under-scoped requests,
- avoid logging raw tokens or sensitive payloads,
- separate internal API behavior from external API contracts.

Related existing documents:

- [Threat model](../threat-model.md)
- [Security boundaries](../security-boundaries.md)
- [Request flow diagrams](../architecture/request-flow.md)
- [External API access model](../architecture/external-api-access-model.md)
- [API key lifecycle](../token-lifecycle/api-key-lifecycle.md)
- [Permission model](../scopes/permission-model.md)
- [External API audit log](../auditability/external-api-audit-log.md)
- [External API rate limiting](../rate-limiting/external-api-rate-limiting.md)
- [Internal vs external API](../integration-boundaries/internal-vs-external-api.md)
