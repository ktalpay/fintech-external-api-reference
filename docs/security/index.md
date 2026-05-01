# Security

This section collects security guidance for external API access.

Core themes:

- resolve company scope from trusted API key metadata,
- store only token hashes,
- deny invalid, expired, revoked, or under-scoped requests,
- avoid logging raw tokens or sensitive payloads,
- separate internal API behavior from external API contracts.

Related existing documents:

- [Security boundaries](../security-boundaries.md)
- [Threat model](../threat-model.md)
