# ADR 004: Business-Hours-Aware Availability

## Status

Accepted

## Context

The service is not expected to run continuously across all days. Treating all downtime as an outage would create false positives and operational noise.

## Decision

Availability semantics are modeled as:

- actual reachability: `service:up:raw:*`
- expected schedule: `service:expected_up:business_hours`
- business-hours-aware availability: `service:availability:business_hours:*`

The currently implemented schedule is:

- timezone: Asia/Taipei
- expected up on weekdays
- expected down on weekends

## Consequences

Positive:

- fewer false alerts
- alert semantics better match operating policy
- dashboards can distinguish outage from expected downtime

Trade-offs:

- availability is policy-aware, not pure uptime
- timezone and schedule are part of the semantic contract and must be documented clearly
