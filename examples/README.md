# Examples

This directory contains minimal, generic examples for the external API reference architecture.

Examples are illustrative only. They are non-production references and do not include real company, customer, or environment details.

## OpenAPI

- [Minimal external API contract](openapi/external-api.sample.yaml)

The sample shows an `X-Api-Key` security scheme, a small set of external endpoints, and standard error responses for unauthorized, forbidden, and rate-limited requests.

Read the sample as an implementation-oriented example of the grouped documentation model: `X-Api-Key` resolves to a token owner, the token owner determines effective company scope, and callers must not choose arbitrary company scope.
