# ADR 001: Metrics-First Observability

## Status

Accepted

## Context

The vLLM service exposes Prometheus metrics directly. The system needs usage accounting, health monitoring, and engineering diagnosis without introducing additional runtime coupling.

## Decision

The repository uses metrics as the primary observability source of truth.

Dashboards, alerts, and validation are built from Prometheus metrics rather than log parsing or external state.

## Consequences

Positive:

- simple runtime architecture
- low operational coupling
- deterministic queries and alerts
- easy local reproducibility

Trade-offs:

- semantics must be encoded carefully in recording rules
- validation must account for counter resets and histogram behavior
- logs remain useful for detailed debugging, but are outside this repository's semantic contract
