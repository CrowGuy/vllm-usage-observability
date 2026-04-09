# Scaling — VLLM Usage Observability

## Overview

This document describes how the observability system behaves as the number of vLLM instances increases, and how to scale it safely.

The system is designed to support:

* horizontal scaling of vLLM instances
* multi-model deployments
* long-term metric retention

---

## Scaling Dimensions

The system scales along three primary axes:

### 1. Number of Instances

* more vLLM containers / machines
* each instance adds:

  * metrics
  * time series
  * scrape load

---

### 2. Number of Models

* multiple models deployed simultaneously
* increases label combinations (`model_name`)

---

### 3. Metric Cardinality

* number of unique label combinations
* directly impacts Prometheus memory usage

---

## What Happens When You Scale

### More Instances → More Time Series

Each new instance adds:

* duplicate metric families (same metric name)
* new label combination (`instance_name`)

Example:

```text id="6z9n1y"
vllm:prompt_tokens_total{instance_name="a"}
vllm:prompt_tokens_total{instance_name="b"}
vllm:prompt_tokens_total{instance_name="c"}
```

---

### Impact on Prometheus

| Component   | Impact                           |
| ----------- | -------------------------------- |
| Memory      | increases with time series count |
| CPU         | increases with query complexity  |
| Disk        | increases with retention         |
| Scrape load | linear with number of targets    |

---

## Label Cardinality Control

### Safe Labels

* `instance_name`
* `model_name`
* `env`
* `region`

---

### Dangerous Labels (Do NOT use)

* `user_id`
* `request_id`
* `session_id`
* timestamps

These create **unbounded cardinality** and will break Prometheus.

---

## Aggregation Strategy (Critical)

The system uses **three aggregation layers**:

| Level    | Purpose    |
| -------- | ---------- |
| instance | debugging  |
| model    | reporting  |
| global   | monitoring |

---

### Why This Matters

Without aggregation:

* dashboards become slow
* queries explode in complexity
* alerts become unreliable

---

### Example

Bad (no aggregation):

```promql id="bad1"
vllm:prompt_tokens_total
```

Good:

```promql id="good1"
usage:tokens:rolling24h:global
```

---

## Prometheus Scaling Limits

### Rule of Thumb

| Scale          | Recommendation              |
| -------------- | --------------------------- |
| ≤ 5 instances  | current setup OK            |
| 5–20 instances | monitor memory closely      |
| 20+ instances  | consider scaling Prometheus |

---

### Key Metrics to Watch

```promql id="k1"
prometheus_tsdb_head_series
```

```promql id="k2"
prometheus_tsdb_head_chunks
```

```promql id="k3"
process_resident_memory_bytes
```

---

## Scrape Strategy

### Current

* static_configs
* fixed targets

---

### When Scaling

You may need:

* service discovery (future)
* dynamic target management

---

## Recording Rules Efficiency

Recording rules reduce query cost by:

* precomputing aggregates
* reducing repeated computation

---

### Important Practice

Always use:

* `usage:*`
* `latency:*`
* `runtime:*`

Avoid querying raw metrics directly in dashboards.

---

## Dashboard Scaling Behavior

### Engineering Dashboard

* scales with number of instances
* each instance adds more lines

Mitigation:

* use `instance_name` filter
* avoid showing too many instances at once

---

### Management Dashboard

* uses aggregated metrics
* remains stable regardless of instance count

---

## Alerting at Scale

### Instance Alerts

* scale linearly with instance count
* more instances → more potential alerts

---

### Global Alerts

* stable regardless of instance count
* used for service-level monitoring

---

### Alert Noise Control

* use `for:` durations (5–30 min)
* avoid reacting to short spikes

---

## Retention Strategy

Prometheus stores time series data locally.

### Key Setting

```yaml id="ret1"
--storage.tsdb.retention.time=15d
```

---

### Trade-offs

| Retention | Impact               |
| --------- | -------------------- |
| longer    | more disk usage      |
| shorter   | less historical data |

---

## When to Scale Prometheus

You should consider scaling Prometheus when:

* memory usage is consistently high (>70%)
* query latency increases
* dashboards become slow
* time series count grows rapidly

---

## Scaling Options

### 1. Vertical Scaling

* more RAM
* more CPU

---

### 2. Horizontal Scaling (Advanced)

* sharded Prometheus
* remote storage (future)

---

## Operational Guidelines

### Adding Instances

* ensure labels are correct
* verify metrics appear
* confirm dashboards update automatically

---

### Monitoring Growth

Track:

```promql id="g1"
count(up{job="vllm"})
```

---

### Detect Cardinality Issues

```promql id="g2"
topk(10, count by (__name__)({__name__=~".+"}))
```

---

## Anti-Patterns

### 1. Querying Raw Metrics in Dashboards

Bad:

```promql id="a1"
vllm:prompt_tokens_total
```

---

### 2. Ignoring Labels in Aggregation

Bad:

```promql id="a2"
sum(vllm:prompt_tokens_total)
```

---

### 3. Mixing Aggregation Levels

Bad:

```promql id="a3"
sum by (instance_name) (...) + sum by (model_name) (...)
```

---

## Future Scaling (Phase 5)

* remote write (Thanos / Cortex / Mimir)
* long-term storage
* anomaly detection
* per-team cost tracking

---

## Summary

Scaling the system requires:

* strict label discipline
* correct aggregation usage
* monitoring Prometheus resource usage

The system is designed to:

* scale horizontally with instances
* remain stable via aggregation layers
* provide consistent observability at any scale

---
