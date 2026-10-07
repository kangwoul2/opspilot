# OpsPilot Design

## 1. Design target

OpsPilot validates one claim: an AI agent may make a bad decision, but the application boundary must still prevent unauthorized data access and unauthorized writes.

The model is treated as untrusted input. Safety-critical decisions remain deterministic Core API logic.

## 2. Service boundary

### Agent Service

Responsibilities:
- accept the user's operator bearer token
- run a small explicit tool-calling loop
- expose only an allow-list of tool functions
- forward the same bearer token to Core
- preserve/generate a correlation ID
- capture prompts in test-only fake providers
- translate Core results into explicit ToolResult states
- propose, but never execute, write actions

Forbidden:
- direct PostgreSQL access
- signing JWTs
- holding a service-admin credential
- approving actions
- deciding customer scope
- generating server idempotency keys

### Core API

Responsibilities:
- verify JWT signature, issuer, audience, expiration
- resolve operator from JWT `sub`
- load authorization/customer scope from the database
- perform scope checks before returning domain data
- mask PII before any agent-facing response
- classify freshness from server-owned timestamps/config
- create PendingAction records
- validate approval integrity
- execute approved same-DB mutations exactly once for the MVP
- write sanitized AuditEvent records

## 3. Authentication and authorization

MVP uses an asymmetric JWT test/dev setup.

- private signing key: test/dev token fixture only
- public verification key: Core API
- Agent Service: receives and forwards bearer token, never receives the private key
- required JWT claims: `iss`, `aud`, `sub`, `exp`
- role/scope is not trusted from LLM output
- customer scope is resolved by Core from database state

This intentionally avoids a shared HMAC secret between Agent and Core because that would allow a compromised Agent to mint identities.

Long-term production direction: token exchange / on-behalf-of delegated token. It is PLANNED, not implemented in the MVP.

## 4. Minimal data model

Six domain entities are allowed.

### Operator
- id
- subject (unique JWT sub)
- display_name
- capability: OPERATOR or APPROVER
- active

### Customer
- id
- external_key
- name
- email
- phone
- updated_at

PII fields exist so masking/leakage can be tested. Agent-facing APIs never return raw values.

### OperatorCustomerScope
A join table, not a separate domain aggregate:
- operator_id
- customer_id
- unique(operator_id, customer_id)

### Service
- id
- customer_id
- service_code
- status
- updated_at
- version

The first write demo will use generic `UPDATE_SERVICE_STATUS`. No telecom-only aggregate is introduced.

### Incident
- id
- customer_id
- service_id nullable
- severity
- status
- summary
- updated_at

### PendingAction
- id UUID, server generated
- requester_operator_id
- approver_operator_id nullable
- action_type
- target_type
- target_id
- canonical_payload JSONB
- payload_hash
- status
- expires_at
- created_at
- executed_at nullable
- version

### AuditEvent
- id
- correlation_id
- actor_operator_id nullable
- event_type
- outcome
- target_type nullable
- target_id nullable
- reason_code nullable
- created_at

Audit details must not contain raw PII or entire request/prompt bodies.

## 5. Freshness

Each Core read response includes:
- `asOf`
- `sourceId`
- freshness classification

Freshness thresholds are server configuration, not model judgment.

The Agent maps Core outcomes to:

```text
OK
NOT_FOUND
FORBIDDEN
UNAVAILABLE
STALE
```

`UNAVAILABLE` must never be converted to "no incident" or "not found".
`STALE` must surface `asOf` and a stale warning.

## 6. API boundary

Paths are versioned under `/api/v1`.

### Agent-facing Core read APIs

```http
GET /api/v1/customers/{customerId}/summary
GET /api/v1/customers/{customerId}/services
GET /api/v1/customers/{customerId}/incidents
```

Rules:
- bearer token required
- Core verifies operator scope before returning data
- agent-facing DTOs contain masked PII only
- 403 means scope/capability denied
- 404 means in-scope resource is absent
- stale data is not represented as 404

### Action APIs

```http
POST /api/v1/actions
GET  /api/v1/actions/{actionId}
POST /api/v1/actions/{actionId}/approve
POST /api/v1/actions/{actionId}/reject
```

`POST /actions`
- requester must have scope for target
- Core creates server-owned action id
- Core canonicalizes payload and calculates hash
- initial state: `PENDING_APPROVAL`

`POST /actions/{id}/approve`
- APPROVER capability required
- approver must differ from requester
- action must not be expired
- client supplies the payload hash it reviewed
- supplied hash must equal stored Core hash
- terminal actions cannot be approved again
- Core owns execution identity/idempotency using action id

Agent Service may call `POST /actions`; it must never call approval endpoints as a privileged principal.

## 7. PendingAction state machine

MVP states:

```text
PENDING_APPROVAL
  | approve + execute successfully
  v
EXECUTED

PENDING_APPROVAL -> REJECTED
PENDING_APPROVAL -> EXPIRED
PENDING_APPROVAL -> FAILED
```

There is intentionally no long-lived externally observable `APPROVED` state in the MVP.

For the first write action, approval validation, target mutation, terminal `EXECUTED` transition, and execution audit record occur in one database transaction under a row lock/compare-and-set guard.

Why:
- avoids an approved-but-not-executed split state for same-DB writes
- concurrent approval requests cannot execute twice
- if DB commit succeeds but HTTP response is lost, a retry sees terminal `EXECUTED` and cannot repeat the mutation

Replay contract:
- first valid approval: 200 and EXECUTED
- later approval/retry after commit: 409 `ACTION_ALREADY_EXECUTED` with action id/state
- caller can GET the action to learn final state
- actual mutation count remains exactly one

This exactly-once claim is limited to the MVP's same-PostgreSQL transaction. External side effects require an outbox/execution ledger design and are not claimed here.

## 8. PII boundary

PII must be removed or masked before data enters an LLM prompt.

Leakage checks cover:
- LLM prompt capture
- application logs
- audit logs/events
- error responses

Tests use unique marker strings and assert marker occurrence count is zero.

## 9. Prompt injection boundary

Customer notes, incident summaries, and future runbook text are untrusted.

The project does not claim to solve prompt injection at the model layer.

Instead, `MaliciousFakeLLM` deliberately follows injected instructions and attempts:
- an out-of-scope read
- an unauthorized write/approval path

Expected protection comes from Core authorization/action policy.

## 10. Correlation ID

- accept safe incoming `X-Correlation-Id` or generate one at Agent boundary
- forward to Core
- include in sanitized audit events and error metadata
- never use correlation ID as an authorization or idempotency decision

## 11. Technology review

Kept because directly needed:
- Spring Boot/JPA/PostgreSQL/Flyway/Testcontainers
- FastAPI + explicit tool loop
- fake LLM providers
- JWT
- Docker Compose
- JUnit/pytest/GitHub Actions

Rejected for MVP because they dilute the new evidence:
- Kafka
- Redis
- MinIO
- Parquet/data lake
- LangGraph
- Prometheus/Grafana
- Next.js
- general RAG platform
- external LLM provider

## 12. Phase 1 acceptance criteria

Phase 1 is Core API only.

Required:
1. Java 17 Spring Boot project boots against PostgreSQL.
2. Flyway creates Operator, Customer, scope join table, Service, Incident, and required support columns.
3. Testcontainers integration tests run without a locally installed PostgreSQL.
4. asymmetric JWT verification is enforced by Core.
5. valid in-scope customer read returns 200.
6. cross-scope customer read returns 403.
7. raw PII markers do not appear in agent-facing response DTOs.
8. read responses expose `asOf`/freshness inputs needed by Phase 2.
9. correlation ID is returned/propagated at Core boundary.
10. CI runs Phase 1 tests.

Not in Phase 1:
- Agent Service
- PendingAction execution
- approval
- malicious fake LLM
- external LLM
- RAG

Phase 1 is complete only when the integration tests pass in CI. Local code existence alone is not completion.
