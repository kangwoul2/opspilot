# Work State

## Current Phase

Phase 0 — COMPLETE.

Phase 1 has not started.

## Verified Commit

`7c1fb535c7123832d60084d5f018ee8303b13a3a`

At this commit, the required Phase 0 files were fetched back from GitHub and confirmed present:
- README.md
- docs/DESIGN.md
- docs/FAILURE_TESTS.md
- docs/WORK_STATE.md
- ADR 001-003
- .github/workflows/ci.yml

The connector's combined commit-status endpoint returned no legacy statuses for this commit. Therefore the workflow file is IMPLEMENTED, but a GitHub Actions runtime pass is not claimed yet.

## Completed

- repository confirmed public and initially empty
- MVP narrowed to Core API + Agent Service
- trust boundary fixed: LLM/Agent is not an authorization authority
- six-domain-entity maximum retained
- asymmetric JWT decision fixed
- same operator identity forwarding chosen for MVP
- Agent receives no signing private key
- operator/customer scope remains Core database state
- API boundary fixed
- PendingAction state machine fixed
- no durable APPROVED intermediate state for same-DB MVP execution
- seven failure-test contracts fixed at test-name/assertion level
- Phase 1 acceptance criteria fixed
- over-scoped technologies explicitly excluded
- Phase 0 contract CI workflow added

## Measured

Repository/file existence at the verified commit was re-read through GitHub.

No runtime application result is claimed yet.
No GitHub Actions pass is claimed yet.

## Known Limitations

- no application code exists yet
- no executable JUnit/pytest tests exist yet
- CI workflow runtime result has not been verified
- no Docker Compose stack exists yet
- no external identity provider
- no delegated token exchange
- exactly-once design is limited to same-PostgreSQL mutations
- no external LLM
- no RAG
- no UI
- no production/security certification claim

## Next Exact Task

Start Phase 1 only after the Phase 0 report.

Phase 1 first task:
create the minimal Java 17 Spring Boot Core API skeleton, Flyway schema, PostgreSQL Testcontainers base integration test, and asymmetric JWT verification fixture.

Then implement only the Core portion required for:
- authorized read -> 200
- cross-scope read -> 403
- PII-masked agent-facing DTO
- asOf/freshness metadata
- correlation ID propagation

Do not create Agent Service yet.
Do not implement PendingAction yet.

## Important Decisions

1. Agent never connects directly to PostgreSQL.
2. Agent never receives a JWT signing private key.
3. Agent forwards the operator bearer token unchanged for the MVP.
4. Core resolves customer authorization from database scope, not model output.
5. Agent can propose an action but cannot approve or execute it.
6. Core canonicalizes payload and computes payload hash.
7. Core-generated PendingAction id is the execution idempotency identity.
8. approval validation + first MVP mutation + terminal state + audit are one DB transaction.
9. terminal action replay is rejected and never re-executes.
10. prompt injection defense claim is boundary containment, not perfect model-level prevention.
