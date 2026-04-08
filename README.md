# VLLM Usage Observability

## Overview

This project provides a **metrics-based observability system** for vLLM inference services.

It enables:

* Accurate **usage accounting** (requests, tokens)
* Reliable **service availability monitoring**
* Engineering-level **performance diagnostics**

The system is designed to be:

* Resilient to **counter resets** (e.g., container restart)
* Aware of **scheduled downtime** (e.g., weekends)
* Independent of raw vLLM metric naming
* Fully reproducible via configuration-as-code

---

## Architecture

```
vLLM (/metrics)
        ↓
Prometheus (scrape + storage)
        ↓
Recording Rules (semantic layer)
        ↓
Grafana (dashboards)
```

---

## Key Concepts

### 1. Metrics over Logs

* Logs are used for debugging
* Metrics are the **source of truth** for usage and observability

---

### 2. Counter-Based Accounting

All usage metrics are derived from monotonic counters:

```
increase(...)
```

This ensures correctness even when:

* vLLM restarts
* counters reset to zero

---

### 3. Canonical Metrics

Dashboards and alerts must use **canonical metrics**, not raw exporter metrics.

Examples:

* `usage:requests:*`
* `usage:tokens:*`
* `service:*`
* `latency:*`

---

### 4. Separation of Concerns

| Category    | Purpose            |
| ----------- | ------------------ |
| Usage       | Business reporting |
| Service     | Availability       |
| Engineering | Debugging          |

---

## Project Structure

```
deploy/
  prometheus/
    prometheus.yml
    recording_rules.yml
  grafana/
    provisioning/
      datasources/
      dashboards/
    dashboards/
      management.json
      engineering.json
```

---

## Getting Started

### 1. Prerequisites

* Docker & Docker Compose
* A running vLLM service exposing `/metrics`

---

### 2. Configure Prometheus Target

Edit:

```
deploy/prometheus/prometheus.yml
```

Set your vLLM host:

```yaml
targets:
  - <VLLM_HOST>:8000
```

---

### 3. Start the Stack

```bash
docker compose up -d
```

---

### 4. Access UI

* Prometheus: http://localhost:9090
* Grafana: http://localhost:3000

Default credentials:

```
admin / admin
```

---

## Validation (Phase 1)

### Important

If Prometheus is newly deployed:

* `rolling24h` metrics will be **0 initially**
* This is expected behavior

---

### Verify Pipeline

Run in Prometheus:

```promql
up{job="vllm"}
```

Expected:

```
1
```

---

### Verify Raw Metrics

```promql
vllm:prompt_tokens_total
```

---

### Verify Short-Term Increase

```promql
increase(vllm:prompt_tokens_total[5m])
```

Expected:

* > 0 after sending requests

---

### Verify Recording Rules

```promql
usage:tokens:increase1h
```

Expected:

* > 0 when traffic exists

---

## Known Behaviors

### 1. Counter Reset on Restart

When vLLM restarts:

* Raw counters reset to 0
* Usage metrics remain correct

---

### 2. Warm-up Period

After Prometheus starts:

* `rolling24h` metrics are not valid immediately
* Requires up to 24 hours of data

---

### 3. Service Downtime

When vLLM stops:

* `service:up:raw = 0`
* `service:availability:business_hours` depends on weekday

---

### 4. Ephemeral Containers

Using:

```bash
docker run --rm
```

is supported.

Effects:

* counters reset ✔
* time-series may fragment ✔
* usage metrics remain correct ✔

---

## Dashboards

### Management Dashboard

Provides high-level business metrics:

* Requests (24h)
* Tokens (24h)
* Usage trends
* Service availability

---

### Engineering Dashboard (Phase 2)

Will include:

* latency breakdown
* queue vs inference time
* cache efficiency
* error rates

---

## Troubleshooting

### Target Down

```promql
up{job="vllm"} == 0
```

Check:

* network connectivity
* port exposure
* firewall rules

---

### No Usage Data

Check:

```promql
increase(vllm:prompt_tokens_total[5m])
```

If 0:

* no traffic
* or scrape not working

---

### Recording Rules Not Working

Check:

```
Prometheus → Status → Rules
```

---

### Dashboard Empty

Check:

* datasource UID = `prometheus`
* Prometheus query returns data

---

## Roadmap

### Phase 1

* Validation & baseline
* README documentation

### Phase 2

* Engineering dashboard

### Phase 3

* Alerting (business-hours aware)

### Phase 4

* Hardening (multi-instance, scaling)

---

## Summary

This project transforms raw vLLM metrics into:

* reliable usage accounting
* actionable observability signals
* scalable monitoring infrastructure

---
