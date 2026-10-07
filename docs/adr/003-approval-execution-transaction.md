# ADR 003: Execute first write atomically with approval validation

Status: Accepted

## Context

A durable APPROVED state followed by a separate same-DB executor creates an unnecessary crash window for this MVP. Concurrent approvals and a lost HTTP response also make duplicate execution a key failure case.

## Decision

For the first generic `UPDATE_SERVICE_STATUS` action:

- lock/CAS the PendingAction
- validate current state, expiry, requester/approver separation, and reviewed payload hash
- apply target mutation
- write execution audit event
- transition action to EXECUTED

in one PostgreSQL transaction.

No externally observable durable APPROVED state is used.

## Consequences

Twenty concurrent approvals cannot apply the same DB mutation more than once.
If commit succeeds and response is lost, a retry observes terminal state and cannot re-execute.

This guarantee is deliberately limited to same-database effects. External APIs would require a different execution ledger/outbox design.
