# Dashboard Philosophy

## Purpose

The dashboards in this repository are not generic metric browsers. They are opinionated views over the semantic contract.

The design intent is:

- Management dashboard answers "how is the service doing?"
- Engineering dashboard answers "which instance is unhealthy and why?"

## Design Principles

### 1. Dashboards Consume Canonical Metrics

Dashboards should prefer recording-rule outputs over raw metrics.

Reason:

- stable semantics
- lower query complexity
- consistent reuse across panels

### 2. Different Audiences Need Different Aggregations

The dashboards intentionally use different aggregation levels:

- Management: mostly `:model` and `:global`
- Engineering: mostly `:instance`

This is a feature, not duplication.

### 3. Management Dashboard Is Service-Oriented

The Management dashboard should favor:

- usage totals
- usage trends
- service availability
- global health signals

It should avoid overwhelming readers with per-node detail.

### 4. Engineering Dashboard Is Diagnosis-Oriented

The Engineering dashboard should favor:

- per-instance filters
- latency decomposition
- backlog and concurrency
- cache behavior
- HTTP error and latency views

It should make partial failures visible quickly.

### 5. Dashboards Are Not the Place to Rebuild Semantics

If a panel requires complex raw PromQL to reconstruct meaning, that is usually a sign the recording rules should express that meaning instead.

## Current Dashboard Roles

### Management Dashboard

Primary questions:

- how much usage did we process?
- how is usage trending?
- is the service broadly healthy?
- how many instances are active?

### Engineering Dashboard

Primary questions:

- which instance is impacted?
- is the issue queueing, latency, cache efficiency, or HTTP-level failure?
- is the issue local or systemic?

## Safe Evolution Guidance

Future dashboard changes should preserve:

- canonical-metric usage
- clear audience separation
- explicit aggregation level
- low cognitive load for first diagnosis

Future dashboard changes should avoid:

- ad hoc raw metric dependence
- mixing management and engineering concerns into one panel set
- hidden business logic inside panel queries
