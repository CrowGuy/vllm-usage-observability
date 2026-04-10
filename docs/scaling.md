# Scaling

## Purpose

This document explains how the semantic observability design behaves as deployment size increases.

The key scaling question is not only "can Prometheus scrape more targets?" It is also "does the semantic model remain clear and affordable as target count grows?"

## Scaling Dimensions

The system scales along these axes:

### Number of Instances

Each new instance adds:

- another scrape target
- another set of raw metric series
- more instance-level canonical outputs

### Number of Models

Each additional model increases:

- model-level aggregation combinations
- usage reporting dimensionality

### Label Cardinality

Cardinality is often the real scaling limit.

Unsafe labels can make the system unusable even at modest instance counts.

## Why the Aggregation Model Matters

The three-layer aggregation design is a scaling control mechanism:

- `:instance` keeps diagnosis explicit
- `:model` keeps reporting stable
- `:global` keeps service monitoring readable

Without this structure:

- dashboard queries become harder to reason about
- alert semantics drift
- PromQL duplication grows

## Prometheus Impact

As instance count increases, expect:

- memory growth from more active series
- CPU growth from more queries and rule evaluation
- disk growth from retention
- linear scrape load growth

Representative Prometheus metrics to watch:

```promql
prometheus_tsdb_head_series
prometheus_tsdb_head_chunks
process_resident_memory_bytes
```

## Labeling Discipline

Safe dimensions in this repository include:

- `instance_name`
- `model_name`
- `env`
- `region`

Avoid unbounded labels such as:

- `user_id`
- `request_id`
- `session_id`
- timestamps

These are incompatible with the current design.

## Dashboard Scaling

### Management Dashboard

The Management dashboard should remain relatively stable as instance count grows because it mostly consumes aggregated signals.

### Engineering Dashboard

The Engineering dashboard naturally grows noisier with instance count because its purpose is diagnosis. Use instance filters aggressively to keep it readable.

## Alert Scaling

### Instance Alerts

Instance alerts scale with fleet size. More instances mean more possible local failures.

### Global Alerts

Global alerts remain fleet-wide signals and should stay comparatively stable as the deployment grows.

## Current Operational Boundary

The current repository uses static Prometheus target configuration.

That is acceptable for the current Phase 4 scope, but future larger-scale deployments may need:

- dynamic target management
- service discovery
- stronger Prometheus capacity planning

These are future concerns, not current repository capabilities.

## Retention

Prometheus retention directly affects disk consumption and query range cost. The current Docker Compose configuration uses 30-day retention.

Longer retention means:

- more disk use
- more historical visibility

Shorter retention means:

- less disk use
- less long-window visibility

## Practical Guidance

As scale increases:

1. preserve stable labels
2. prefer canonical metrics over raw queries
3. keep management and engineering views separate
4. watch Prometheus resource metrics
5. resist introducing high-cardinality labels
