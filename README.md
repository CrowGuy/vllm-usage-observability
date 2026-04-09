# VLLM Usage Observability

## Project Overview

VLLM Usage Observability is a metrics-based observability stack for vLLM inference services. It provides:

- usage accounting from Prometheus counters
- service availability monitoring
- engineering diagnostics for latency, queueing, cache behavior, and HTTP error rates
- dashboards and alerting built on canonical recording rules rather than raw exporter metrics

The project is currently implemented through Phase 4:

- Phase 1: metrics definition, validation, and the core observability pipeline
- Phase 2: Management and Engineering Grafana dashboards
- Phase 3: alerting, including business-hours-aware availability semantics
- Phase 4: multi-instance support across Prometheus targets, recording rules, dashboards, and alerts

This repository does not currently implement:

- Alertmanager or Slack integration
- anomaly detection
- cost attribution

## Architecture Overview

```text
Multiple vLLM Instances (/metrics)
        ↓
Prometheus (scrape + storage)
        ↓
Recording Rules (semantic layer)
        ↓
Alerting Rules (evaluation layer)
        ↓
Grafana (visualization)
```

High-level behavior:

- vLLM instances expose raw Prometheus metrics at `/metrics`
- Prometheus scrapes those targets and attaches stable labels such as `instance_name`, `env`, and `region`
- recording rules produce canonical metrics for dashboards and alerts
- alerting rules evaluate those canonical metrics
- Grafana provides a management view and an engineering view

For architecture details, see `docs/architecture.md`.

## Key Concepts

### Metrics Over Logs

Metrics are the source of truth for usage accounting, service health, and operational monitoring. Logs may still help with debugging, but dashboards and alerts in this project are intentionally derived from Prometheus metrics.

### Canonical Metrics (Recording Rules)

Dashboards and alerts should use canonical metrics produced by `deploy/prometheus/recording_rules.yml`, not raw vLLM metrics directly.

Examples:

- `usage:*` for requests and tokens
- `runtime:*` for concurrency, queueing, and cache behavior
- `latency:*` for p95 and p99 performance signals
- `service:*` for availability and business-hours semantics
- `http:*` for request rates, status codes, and error rates

This creates a stable semantic layer that is easier to query and less sensitive to raw metric naming changes.

### Business-Hours-Aware Availability

Availability is not treated as "the service must always be up." The current implementation defines expected availability using:

- `service:business_day:asia_taipei`
- `service:expected_up:business_hours`
- `service:availability:business_hours:*`

Operationally, this means:

- weekdays are considered expected service days
- weekends are treated as scheduled downtime
- availability alerts are suppressed during expected weekend shutdowns

## Project Structure

```text
.
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
    ├── alerting.md
    ├── architecture.md
    ├── labeling.md
    ├── metrics.md
    ├── runbook.md
    └── scaling.md
```

Primary files:

- `docker-compose.yml`: local Prometheus and Grafana stack
- `deploy/prometheus/prometheus.yml`: scrape targets and target labels
- `deploy/prometheus/recording_rules.yml`: canonical metrics
- `deploy/prometheus/alerts.yml`: Prometheus alert rules
- `deploy/grafana/dashboards/management.json`: high-level usage and service dashboard
- `deploy/grafana/dashboards/engineering.json`: per-instance operational dashboard

## Getting Started

### Prerequisites

- Docker and Docker Compose
- one or more running vLLM instances exposing `/metrics`

### Configure Prometheus Targets

Edit `deploy/prometheus/prometheus.yml` and update `static_configs` under the `vllm` job to match your environment.

Each target should have stable labels. The current configuration uses:

- `service`
- `component`
- `env`
- `region`
- `instance_name`

`model_name` is part of the documented labeling strategy and is recommended for model-level aggregation. Make sure your scrape targets and exposed metrics provide the labels needed for your deployment.

Example target block:

```yaml
- targets:
    - 10.0.0.11:8000
  labels:
    service: vllm
    component: inference
    env: prod
    region: tw
    instance_name: gpt-oss-120b-a
```

### Start the Stack

```bash
docker compose up -d
```

### Access Prometheus and Grafana

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

Default Grafana credentials:

```text
admin / admin
```

The Management dashboard is configured as the default Grafana home dashboard.

## Validation & Known Behaviors

### Basic Validation

After the stack is running, validate the pipeline in Prometheus:

```promql
up{job="vllm"}
```

Expected result:

- `1` for reachable targets

Validate raw counters:

```promql
vllm:prompt_tokens_total
vllm:generation_tokens_total
vllm:request_success_total
```

Validate canonical metrics:

```promql
usage:tokens:increase1h:global
service:up:raw:instance
latency:e2e:p95:global
```

### Counter Reset Behavior

Usage accounting is derived from counters with `increase()` and rates with `rate()`. If a vLLM instance restarts and raw counters reset to zero:

- raw counters will drop
- time series may split across restarts
- canonical usage metrics remain valid over the queried time window

This is a core design assumption from `docs/metrics.md`.

### Rolling 24h Warm-Up

Metrics such as `usage:requests:rolling24h:*` and `usage:tokens:rolling24h:*` are not meaningful immediately after Prometheus starts.

Expected behavior:

- values may be `0` or lower than expected at first
- full rolling-24-hour views require up to 24 hours of retained samples

### Weekend Shutdown Semantics

The current implementation encodes business-day semantics in Asia/Taipei time:

- Monday through Friday: service is expected to be up
- Saturday and Sunday: service is treated as expected downtime

As a result:

- `service:expected_up:business_hours` is `0` on weekends
- business-hours availability alerts do not fire during weekend shutdowns
- weekend downtime should not be interpreted as an outage by this system

### Impact of `docker run --rm`

Ephemeral vLLM containers are supported from an accounting perspective, but there are operational consequences:

- container restarts reset raw counters
- Prometheus may observe fragmented time series
- short-lived containers can make debugging more difficult
- canonical usage metrics remain correct if labels stay stable and Prometheus sees the samples

## Dashboards

### Management Dashboard

File:

- `deploy/grafana/dashboards/management.json`

Purpose:

- business and service overview
- model-level usage reporting
- service-wide health signals

Current coverage includes:

- requests, prompt tokens, generation tokens, and total tokens over 24 hours
- hourly usage trends
- usage by model
- business-hours-aware service availability
- active instance count
- global error rate and global end-to-end latency

Use this dashboard first when you need a high-level answer to "how is the overall service behaving?"

### Engineering Dashboard

File:

- `deploy/grafana/dashboards/engineering.json`

Purpose:

- per-instance debugging and operational analysis
- latency and queue investigation
- cache and HTTP behavior inspection

Current coverage includes:

- requests running and waiting
- KV cache usage and prefix cache hit rate
- selected instance availability
- per-instance concurrency
- latency overview and latency breakdown
- HTTP chat request rate by status
- HTTP chat latency p95

Use this dashboard when you need to identify which instance is unhealthy and why.

## Alerting

Alerting is implemented in `deploy/prometheus/alerts.yml` and described in `docs/alerting.md`. Alerts are built on canonical metrics and split into instance and global scopes.

Current alert categories include:

- availability
- API error rate
- latency
- queue and backlog
- cache efficiency
- traffic drop during business hours

Examples of implemented alerts:

- `VLLMInstanceDownBusinessHours`
- `VLLMAllInstancesDownBusinessHours`
- `VLLMChatErrorRateHighInstance`
- `VLLMChatErrorRateHighGlobal`
- `VLLME2ELatencyP95HighInstance`
- `VLLME2ELatencyP95HighGlobal`
- `VLLMQueueTimeP95HighInstance`
- `VLLMQueueTimeP95HighGlobal`

Business-hours awareness is part of alert evaluation, so weekend shutdowns do not create false availability pages.

For response procedures, see `docs/runbook.md`.

## Multi-Instance Support

Phase 4 adds multi-instance support across scraping, recording rules, dashboards, and alerts.

Design summary:

- Prometheus scrapes multiple vLLM targets
- each target is identified with stable labels, especially `instance_name`
- recording rules generate canonical metrics at three aggregation levels:
  - `:instance`
  - `:model`
  - `:global`
- dashboards use the aggregation level appropriate to the audience
- alerts are split into instance and global scopes

Typical usage:

- `:instance` for node-level debugging
- `:model` for usage reporting across instances serving the same model
- `:global` for service-wide monitoring

For labeling details, see `docs/labeling.md`.

## Scaling Notes

The system is designed to scale horizontally with additional vLLM instances, but Prometheus cost grows with:

- number of instances
- number of models
- label cardinality
- retention period

Important scaling constraints from `docs/scaling.md`:

- keep labels stable and low cardinality
- avoid labels such as `user_id`, `request_id`, `session_id`, or timestamps
- prefer canonical recording rules in dashboards instead of raw metric queries
- expect instance-level dashboards and alerts to scale roughly with instance count
- expect Prometheus memory, CPU, and disk use to grow with time series count and retention

The current setup is still based on static Prometheus targets. Dynamic service discovery is future work, not part of the implemented Phase 4 scope.

## Troubleshooting

### Prometheus shows `up{job="vllm"} = 0`

Check:

- the vLLM process is running
- `/metrics` is reachable from Prometheus
- target addresses in `deploy/prometheus/prometheus.yml` are correct
- target labels are valid and consistent

### Management dashboard shows zero or low 24h usage after startup

This is usually normal during the rolling-window warm-up period. Wait for enough history to accumulate before treating it as a data problem.

### Dashboards look inconsistent across instances

Check labeling first:

- `instance_name` should be stable and unique
- `env` and `region` should be set consistently
- `model_name` should be present where model-level aggregation is expected

### Availability alert behavior looks wrong on weekends

Verify the intended semantics before changing anything:

- this project currently treats weekends as scheduled downtime
- the business-day rule is encoded in Asia/Taipei time

### Counters appear to reset unexpectedly

This usually indicates a vLLM restart or an ephemeral container lifecycle. Confirm the restart event before treating the metric drop as data corruption.

For operational alert response flows, see `docs/runbook.md`.

## Roadmap / Future Work

The following items are not implemented in this repository today:

- Alertmanager integration
- Slack or other notification routing
- anomaly detection
- cost attribution
- dynamic service discovery for large-scale target management

Any future work should preserve the current design principles:

- metrics as the source of truth
- canonical recording rules as the semantic layer
- stable, low-cardinality labels
- clear separation between instance, model, and global aggregation
