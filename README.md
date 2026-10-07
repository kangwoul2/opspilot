# OpsPilot

OpsPilot is a small service-operations reference project for testing how an AI agent behaves when connected to real application boundaries.

The project does **not** assume the model is trustworthy. The core question is:

> What happens when an AI agent follows the wrong instruction?

The MVP keeps authorization, data scope, write execution, approval, PII handling, freshness decisions, and auditability in a Spring Boot Core API. A FastAPI agent can call narrow tools, but it cannot access PostgreSQL directly and does not hold write credentials.

## MVP

- Java 17, Spring Boot, Spring Data JPA
- PostgreSQL, Flyway, Testcontainers
- Python, FastAPI
- Small explicit tool-calling loop
- Scripted and malicious fake LLM providers
- Asymmetric JWT verification at the Core boundary
- Operator-to-customer scope enforcement
- PII masking before agent/LLM exposure
- Explicit tool result states: OK, NOT_FOUND, FORBIDDEN, UNAVAILABLE, STALE
- PendingAction + human approval + deterministic payload hash
- Exactly-once-in-MVP execution for same-DB writes
- Correlation ID and audit events
- JUnit, pytest, GitHub Actions, Docker Compose

## Non-goals for the MVP

Kafka, Redis, MinIO, Parquet, LangGraph, Prometheus, Grafana, Next.js, a general RAG platform, a data lake, and a production identity provider are intentionally excluded.

## Trust boundary

```text
Client
  |
  | operator JWT
  v
FastAPI Agent
  |
  | forwards same operator identity
  | narrow read tools / propose_action only
  v
Spring Boot Core API
  |
  | authorization / customer scope / masking / freshness
  | pending action / approval / executor / audit
  v
PostgreSQL
```

The agent never connects to PostgreSQL. The agent never receives a signing key. Authorization decisions are made by the Core API, not by the model.

## Phase 0 status

Phase 0 defines the acceptance contract before feature implementation:

- failure tests and assertions: `docs/FAILURE_TESTS.md`
- architecture, data model, APIs, auth, action state machine: `docs/DESIGN.md`
- resumable development state: `docs/WORK_STATE.md`
- decision records: `docs/adr/`

Phase 1 implementation must not expand scope beyond those contracts without updating an ADR.

## Evidence labels

Portfolio claims must be tagged mentally as one of:

- IMPLEMENTED
- MEASURED
- STUDIED
- PLANNED

Code existence is not the same as a passing test, integration verification, or production experience.

## Current phase

**Phase 0 — scope and acceptance criteria.**

No feature implementation is claimed yet.
