# Semantic Contract

## Purpose

This document defines the semantic interface of the observability system.

The key idea is simple:

- raw metrics are inputs
- canonical metrics are the contract
- dashboards, alerts, and runbooks consume the contract

This keeps the repository stable even when raw exporter details are noisy, inconvenient, or subject to change.

## Contract Boundary

### Inputs

The system consumes:

- raw vLLM metrics from `/metrics`
- Prometheus scrape status
- HTTP request metrics exposed by the service

### Semantic Layer

The semantic layer is implemented in `deploy/prometheus/recording_rules.yml`.

It defines five canonical metric families:

- `usage:*`
- `runtime:*`
- `latency:*`
- `service:*`
- `http:*`

### Consumers

The following should consume canonical metrics by default:

- Grafana dashboards
- Prometheus alerts
- validation procedures
- operational diagnostics

Direct raw-metric queries are acceptable for input validation, but should not become the normal dashboard or alert surface.

## Design Guarantees

The semantic contract is designed to provide these guarantees:

1. Usage accounting is counter-safe.
2. Request accounting excludes aborted requests.
3. Dashboards and alerts do not need to embed raw metric quirks.
4. Aggregation semantics are explicit and reusable.
5. Availability semantics reflect expected operating schedule.
6. Multi-instance deployments share one contract shape.

## Canonical Families

### Usage

Business accounting metrics built from monotonic counters using `increase()` and `rate()`.

Representative outputs:

- `usage:requests:*`
- `usage:prompt_tokens:*`
- `usage:generation_tokens:*`
- `usage:tokens:*`

### Runtime

Operational state metrics for concurrency, queueing, cache efficiency, and preemptions.

Representative outputs:

- `runtime:requests_running:*`
- `runtime:requests_waiting:*`
- `runtime:kv_cache_usage_perc:*`
- `runtime:prefix_cache_hit_rate5m:*`

### Latency

Percentile-based latency metrics built from histogram buckets.

Representative outputs:

- `latency:ttft:*`
- `latency:itl:*`
- `latency:e2e:*`
- `latency:queue_time:*`

### Service

Availability semantics built from scrape state plus the business-day rule.

Representative outputs:

- `service:up:raw:*`
- `service:expected_up:business_hours`
- `service:availability:business_hours:*`

### HTTP

API-facing request and latency signals derived from HTTP metrics.

Representative outputs:

- `http:requests:rate5m:*`
- `http:chat_completions:requests:rate5m:*`
- `http:chat_completions:error_rate5m:*`
- `http:chat_completions:latency:p95:*`

## Aggregation Contract

The canonical layer uses three aggregation suffixes:

- `:instance`
- `:model`
- `:global`

Interpretation:

- `:instance`: one operational unit
- `:model`: usage grouped by served model
- `:global`: service-level signal

Not every family exposes every suffix. The important contract is that where a suffix exists, it has consistent meaning across the repository.

## Contract Rules

1. Do not rename canonical metrics casually.
2. Do not bypass canonical metrics in dashboards and alerts unless validating inputs.
3. Do not introduce unbounded-cardinality labels into canonical outputs.
4. Do not change aggregation meaning without updating ADRs and contract docs.
5. Do not redefine business-hours semantics implicitly in dashboards or alerts.

## Non-Goals

This semantic contract does not attempt to define:

- cost attribution
- anomaly detection
- notification routing
- external incident-management workflows
