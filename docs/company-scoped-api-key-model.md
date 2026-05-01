# Company-scoped API key model

## API key ownership model
Each API key is owned by exactly one company. Ownership is established at issuance and recorded in a token registry. A key cannot be shared across multiple companies.

Key metadata should include:
- token identifier (non-secret reference),
- token hash/fingerprint,
- owning `companyId`,
- allowed scopes/permissions,
- issue and expiry timestamps,
- status (active/revoked/expired),
- last-used timestamp.

## Token generation
Tokens should be generated with high entropy from a cryptographically secure source. Token presentation format can be human-manageable, but entropy and unpredictability are security-critical.

## Token hashing
Raw API keys should not be stored in plaintext. Persist a one-way hash or equivalent fingerprint representation suitable for lookup and verification.

## CompanyId resolution
When a request includes `X-Api-Key`, the platform resolves `companyId` exclusively from token ownership metadata in the registry.

## Token expiration
Tokens should have explicit expiration to reduce long-term risk from dormant or leaked credentials. Expiry should be enforced at request time.

## Revocation
Revocation should be immediate from the authorization perspective. Once revoked, a token must fail authorization without relying on delayed operational workflows.

## Last-used tracking
Update last-used metadata on successful authentication/authorization attempts (or with clearly defined policy) to support security monitoring, key rotation programs, and incident response.

## Do not trust incoming companyId with API key auth
When `X-Api-Key` is the authentication mechanism, caller-provided `companyId` fields in path/query/body are untrusted for authorization purposes. They may be accepted as business parameters only if they match platform-resolved ownership and endpoint rules.

Authoritative rule:
- **Authorization company context = company mapped from token**.

## Example request behavior
- Token `tk_live_...` belongs to `companyId = 12`.
- Client requests bank list endpoint.
- Platform resolves company from token as `12`.
- Service returns only banks visible to `companyId = 12`.
- Any attempt to request data for `companyId != 12` is denied or ignored according to endpoint contract.
