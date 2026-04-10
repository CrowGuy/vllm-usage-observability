# ADR 003: Three-Layer Aggregation

## Status

Accepted

## Context

The repository needs to support both single-instance diagnosis and multi-instance service views. Ad hoc aggregation inside panels or alerts would create inconsistency over time.

## Decision

The canonical metric model uses three explicit aggregation layers:

- `:instance`
- `:model`
- `:global`

These layers separate:

- diagnosis
- reporting
- service-level monitoring

## Consequences

Positive:

- consistent query shapes
- clear dashboard responsibilities
- explicit alert scope
- scalable multi-instance semantics

Trade-offs:

- more recording rules
- a stronger need for stable labels
- not every family naturally maps to every suffix, so documentation must be explicit
