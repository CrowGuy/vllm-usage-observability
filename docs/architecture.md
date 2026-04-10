# Architecture

## Purpose

This repository implements a semantic observability architecture for vLLM. Its purpose is to turn raw metrics into stable operational meaning.

## System Shape

```text
Multiple vLLM Instances (/metrics)
        ↓
Prometheus (scrape + storage)
        ↓
Canonical Recording Rules
        ↓
Alerts + Grafana
```

The important architectural boundary is between:

- raw input metrics
- canonical semantic metrics

Everything downstream should prefer the canonical layer.

## Components

### vLLM Instances

Each vLLM instance:

- exposes `/metrics`
- runs independently
- contributes usage, runtime, latency, cache, and HTTP signals

### Prometheus

Prometheus is responsible for:

- scraping multiple targets
- attaching stable labels
- storing time series
- evaluating recording and alert rules

### Canonical Recording Rules

The recording rules define the semantic contract. They transform raw inputs into reusable metrics across five families:

- `usage:*`
- `runtime:*`
- `latency:*`
- `service:*`
- `http:*`

This is the core design decision in the repository.

### Alerts

Alerts evaluate canonical metrics only. This keeps alerting logic aligned with the semantic contract rather than raw metric quirks.

### Grafana

Grafana renders two audience-specific views:

- Management dashboard for service and usage overview
- Engineering dashboard for per-instance diagnosis

## Multi-Instance Model

The architecture is explicitly multi-instance.

```text
[vLLM A]   [vLLM B]   [vLLM C]
   ↓           ↓           ↓
        Prometheus
             ↓
    Canonical Recording Rules
   (instance / model / global)
             ↓
     Alerts + Dashboards
```

Correctness depends on stable labels, especially:

- `instance_name`
- `model_name`
- `env`
- `region`

## Aggregation Layers

The semantic layer uses three aggregation levels:

### Instance

Suffix:

```text
*:instance
```

Purpose:

- debugging
- local failure isolation
- per-node latency and backlog analysis

### Model

Suffix:

```text
*:model
```

Purpose:

- usage reporting by model
- cross-instance aggregation for the same served model

### Global

Suffix:

```text
*:global
```

Purpose:

- service-level health
- management summaries
- global alerting

## Business-Hours-Aware Availability

Availability is modeled as a combination of:

- scrape reachability
- expected operating schedule

The implemented schedule is a business-day rule in Asia/Taipei time. This avoids interpreting planned weekend shutdowns as outages.

## Data Flow

1. vLLM emits raw Prometheus metrics.
2. Prometheus scrapes targets and attaches stable labels.
3. Recording rules derive canonical semantic metrics.
4. Alerts and dashboards consume canonical metrics.
5. Runbooks and validation procedures reason about those canonical outputs.

## Architectural Non-Goals

This repository does not currently implement:

- notification routing
- anomaly detection
- cost attribution
- dynamic service discovery

Those are intentionally outside the current Phase 4 scope.
