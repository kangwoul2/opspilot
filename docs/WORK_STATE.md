# Work State

## Current Phase

Phase 0 — COMPLETE pending CI confirmation of this documentation commit set.

## Verified Commit

Set this to the latest Phase 0 commit after repository writes and CI status are checked.

## Completed

- repository confirmed public and initially empty
- MVP narrowed to Core API + Agent Service
- trust boundary fixed: LLM/Agent is not an authorization authority
- six-entity maximum retained
- asymmetric JWT decision fixed
- same operator identity forwarding chosen for MVP
- operator/customer scope remains Core database state
- API boundary fixed
- PendingAction state machine fixed
- no durable APPROVED intermediate state for same-DB MVP execution
- seven failure-test contracts fixed
- Phase 1 acceptance criteria fixed
- over-scoped technologies explicitly excluded

## Measured

None yet. Phase 0 contains design contracts, not runtime evidence.

## Known Limitations

- no application code exists yet
- no tests have run yet
- no Docker Compose stack exists yet
- no external identity provider
- no delegated token exchange
- exactly-once design is limited to same-PostgreSQL mutations
- no external LLM
- no RAG
- no UI
- no production/security certification claim

## Next Exact Task

Start Phase 1 only after Phase 0 report.

Phase 1 first task:
create the minimal Java 17 Spring Boot Core API skeleton, Flyway schema, PostgreSQL Testcontainers base integration test, and asymmetric JWT verification fixture.

Do not create Agent Service yet.

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
