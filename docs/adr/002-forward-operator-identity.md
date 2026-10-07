# ADR 002: Forward operator identity for the MVP

Status: Accepted

## Context

A production on-behalf-of/token-exchange design would add identity infrastructure that is not necessary to prove the core trust-boundary behavior in a short MVP.

A shared HMAC secret in Agent would be unsafe evidence because a compromised Agent could mint identities.

## Decision

Use asymmetric JWTs.

- test/dev fixture owns the private signing key
- Core owns only the public verification key
- Agent receives and forwards the original operator bearer token
- Core verifies issuer, audience, subject, expiry
- Core resolves authorization scope from DB state

## Consequences

The MVP preserves end-user identity at Core without giving Agent token-minting power.

Token exchange/on-behalf-of is documented as a later production direction, not implemented evidence.
