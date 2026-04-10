# ADR 002: Canonical Recording Rules as the Semantic Interface

## Status

Accepted

## Context

Raw metrics are useful but not stable enough to serve directly as the interface for dashboards and alerts. Raw queries also duplicate business logic and increase drift risk.

## Decision

The repository defines a canonical semantic layer in Prometheus recording rules.

Dashboards and alerts should consume canonical metrics such as:

- `usage:*`
- `runtime:*`
- `latency:*`
- `service:*`
- `http:*`

## Consequences

Positive:

- one place to encode semantics
- simpler dashboard queries
- more consistent alert definitions
- better separation between input metrics and operational meaning

Trade-offs:

- recording rules become a critical interface and must be versioned carefully
- changes to canonical meaning require explicit documentation and review
