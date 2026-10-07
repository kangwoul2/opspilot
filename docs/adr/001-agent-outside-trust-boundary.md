# ADR 001: Treat Agent and LLM as outside the authorization trust boundary

Status: Accepted

## Context

Prompt injection or model error can cause a tool call the application did not intend.

## Decision

Authorization, operator/customer scope, PII filtering, freshness classification, action policy, and write execution are enforced by Core API.

Agent:
- has no direct DB access
- has no privileged service credential
- cannot approve actions

## Consequences

A malicious fake model can be allowed to behave badly in tests without relying on model refusal for protection.

This does not claim that prompts themselves are safe. It claims the application boundary limits consequences.
