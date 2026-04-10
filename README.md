# VLLM Usage Observability

## Why This System Exists

vLLM exposes useful Prometheus metrics, but raw exporter metrics are not a sufficient contract for production observability on their own. This repository exists to provide a stable semantic layer for:

- usage accounting
- service health monitoring
- engineering diagnostics
- multi-instance aggregation
- business-hours-aware alerting

The design goal is not "collect as many metrics as possible." The design goal is to turn raw vLLM and HTTP metrics into canonical, queryable signals that engineers can use consistently across dashboards, validation, and alerts.

## Design Summary

This repository implements a metrics-first observability stack for vLLM:

```text
vLLM /metrics
    -> Prometheus scrape and storage
    -> canonical recording rules
    -> alerts and dashboards
```

The current implemented scope is Phase 4:

- validated raw metric mapping
- working Prometheus and Grafana stack
- Management and Engineering dashboards
- canonical alerting
- business-hours-aware availability semantics
- multi-instance support with instance, model, and global aggregation

The runtime architecture is intentionally simple:

- Prometheus scrapes raw metrics
- recording rules define the semantic contract
- Grafana and alerts consume canonical metrics

## Core Design Invariants

The following invariants define the system and should be preserved in future changes:

1. Metrics are the source of truth for observability in this repository.
2. Canonical metrics, not raw exporter metrics, are the public semantic interface for dashboards and alerts.
3. Usage accounting must be counter-safe across restarts and counter resets.
4. Availability semantics are business-hours aware, not "24/7 always expected up."
5. Stable labels are required for correct aggregation and alert scope.
6. Aggregation is explicit and three-layered: `:instance`, `:model`, `:global`.
7. Management views prefer aggregated service-level signals.
8. Engineering views prefer instance-level diagnosis.
9. Runtime semantics must not depend on log parsing or external state.
10. PromQL semantics are encoded in recording rules and should not be reimplemented ad hoc in dashboards.

## Semantic Pipeline

The repository defines a semantic pipeline rather than a loose collection of dashboards:

### 1. Raw Metric Emission

vLLM exposes:

- usage counters
- runtime gauges
- latency histograms
- HTTP request metrics

### 2. Scrape and Label Attachment

Prometheus scrapes one or more vLLM targets and attaches stable dimensions such as:

- `instance_name`
- `env`
- `region`
- `service`
- `component`

### 3. Canonical Recording Rules

`deploy/prometheus/recording_rules.yml` transforms raw metrics into canonical metrics in five semantic families:

- `usage:*`
- `runtime:*`
- `latency:*`
- `service:*`
- `http:*`

### 4. Consumption Layer

Everything above the scrape layer should preferentially use canonical metrics:

- Grafana dashboards
- Prometheus alerts
- validation queries
- operational runbooks

For the formal contract, see `docs/semantic-contract.md`.

## Aggregation Model

The repository uses three explicit aggregation layers:

### `:instance`

Used for:

- node-level debugging
- partial failure detection
- latency and backlog diagnosis

### `:model`

Used for:

- usage reporting by model
- cross-instance model aggregation
- management views of model consumption

### `:global`

Used for:

- service-level health
- global alerting
- high-level management summaries

This model is deliberate. It prevents dashboards and alerts from encoding one-off aggregation logic that drifts over time.

## Canonical Metric Families

### `usage:*`

Canonical business accounting metrics derived from vLLM counters. These include:

- request rates
- hourly increases
- rolling 24-hour usage
- prompt, generation, and total token views

### `runtime:*`

Canonical operational state metrics derived from gauges and counters. These include:

- requests running
- requests waiting
- KV cache usage
- prefix cache rates and hit rate
- preemptions

### `latency:*`

Canonical latency distributions derived from histogram buckets. These include:

- TTFT
- ITL
- end-to-end latency
- queue time
- prefill, inference, and decode latency

### `service:*`

Canonical availability semantics derived from scrape state and business-day rules. These include:

- raw up status
- expected up during business hours
- business-hours-aware availability

### `http:*`

Canonical API metrics derived from HTTP request counters and histograms. These include:

- request rates
- chat completion request rates
- chat completion error rate
- chat completion latency p95

## Business-Hours Availability Model

This repository does not model availability as unconditional uptime.

Instead, it models:

- whether the service is reachable
- whether the service is expected to be running
- whether the service is available during expected business hours

The implemented business-day rule is:

- timezone: Asia/Taipei
- expected up: Monday through Friday
- expected down: Saturday and Sunday

This distinction matters because weekend shutdowns are treated as scheduled downtime, not outages. Availability alerts therefore consume:

- `service:expected_up:business_hours`
- `service:up:raw:*`

For the decision record behind this, see `docs/adr/004-business-hours-availability.md`.

## Repository Structure

```text
.
├── README.md
├── docker-compose.yml
├── deploy/
│   ├── grafana/
│   │   ├── dashboards/
│   │   │   ├── engineering.json
│   │   │   └── management.json
│   │   └── provisioning/
│   │       ├── dashboards/
│   │       └── datasources/
│   └── prometheus/
│       ├── alerts.yml
│       ├── prometheus.yml
│       └── recording_rules.yml
└── docs/
    ├── adr/
    ├── architecture.md
    ├── dashboard-philosophy.md
    ├── labeling.md
    ├── metrics.md
    ├── runbook.md
    ├── scaling.md
    ├── semantic-contract.md
    └── validation.md
```

Key responsibilities:

- `deploy/prometheus/prometheus.yml`: scrape topology and target labeling
- `deploy/prometheus/recording_rules.yml`: semantic contract implementation
- `deploy/prometheus/alerts.yml`: alerting over canonical metrics
- `deploy/grafana/dashboards/management.json`: management-oriented dashboard
- `deploy/grafana/dashboards/engineering.json`: engineering-oriented dashboard

## What This Repository Is / Is Not

This repository is:

- a semantic observability layer for vLLM
- a Prometheus-first implementation
- a multi-instance metrics architecture through Phase 4
- a reference for canonical dashboards, alerts, and validation

This repository is not:

- a general logging platform
- a cost attribution system
- an anomaly detection system
- an Alertmanager integration repository
- a Phase 5 implementation

Not implemented today:

- Alertmanager integration
- Slack integration
- anomaly detection
- cost attribution
- dynamic service discovery

## Getting Started

### Prerequisites

- Docker and Docker Compose
- one or more running vLLM instances exposing `/metrics`

### Configure Targets

Edit `deploy/prometheus/prometheus.yml` and update the `vllm` `static_configs` to match your deployment.

Use stable labels for each target. The current repository expects dimensions such as:

- `instance_name`
- `env`
- `region`
- `service`
- `component`

`model_name` remains an important aggregation dimension and should be present where model-level aggregation is expected.

### Start the Stack

```bash
docker compose up -d
```

### Access the UIs

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

Default Grafana credentials:

```text
admin / admin
```

The Management dashboard is configured as the Grafana home dashboard.

## Validation Model

Validation in this repository follows the semantic pipeline:

1. validate raw metrics exist
2. validate scrape health
3. validate canonical metrics are populated
4. validate expected behavior under resets and warm-up windows
5. validate dashboards and alerts against canonical metrics, not raw queries

Important known behaviors:

- counter resets are expected across restarts and handled through `increase()` and `rate()`
- rolling 24-hour metrics require warm-up time after Prometheus starts
- weekend downtime is expected by design
- ephemeral containers created with `docker run --rm` can fragment time series while preserving accounting correctness if labels remain stable

See `docs/validation.md` for the full validation model.

## Further Reading

- `docs/architecture.md`
- `docs/semantic-contract.md`
- `docs/metrics.md`
- `docs/labeling.md`
- `docs/dashboard-philosophy.md`
- `docs/validation.md`
- `docs/runbook.md`
- `docs/scaling.md`
- `docs/adr/001-metrics-first.md`
- `docs/adr/002-canonical-recording-rules.md`
- `docs/adr/003-three-layer-aggregation.md`
- `docs/adr/004-business-hours-availability.md`
