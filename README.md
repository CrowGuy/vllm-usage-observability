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

### Engineering Dashboard

The Engineering Dashboard provides **real-time diagnostic visibility** into the vLLM service.

Unlike the Management Dashboard (which focuses on usage and business metrics), this dashboard is designed to help engineers answer:

* Is the system under load?
* Where is latency coming from?
* Is the cache working effectively?
* Are API errors increasing?
* Is the service healthy?

---

#### Key Sections

##### 1. Runtime / Concurrency

Panels:

* Requests Running
* Requests Waiting
* Request Concurrency (time series)

How to interpret:

* `running ↑` → system is actively processing
* `waiting ↑` → queue is building → potential overload
* `waiting + latency ↑` → queue bottleneck

---

##### 2. Latency Overview

Panels:

* E2E latency p95
* TTFT p95 (time to first token)
* ITL p95 (inter-token latency)

How to interpret:

* **E2E ↑** → overall user experience degraded
* **TTFT ↑** → prompt processing / prefill issue
* **ITL ↑** → token generation slowdown

---

##### 3. Latency Breakdown

Panels:

* Queue time p95
* Prefill time p95
* Decode time p95
* Inference time p95

How to interpret:

| Symptom     | Likely Cause                  |
| ----------- | ----------------------------- |
| queue ↑     | too many concurrent requests  |
| prefill ↑   | long prompts / CPU bottleneck |
| decode ↑    | GPU throughput issue          |
| inference ↑ | overall compute slowdown      |

---

##### 4. Cache Efficiency

Panels:

* KV Cache Usage
* Prefix Cache Hit Rate

How to interpret:

* **KV cache near 1.0** → memory pressure risk
* **low hit rate** → cache ineffective → higher latency
* **high hit rate** → reuse working → better performance

---

##### 5. HTTP / API Layer

Panels:

* Requests by status
* Error rate (5m)
* HTTP latency p95

How to interpret:

* **4xx ↑** → client issues (bad request)
* **5xx ↑** → server issues
* **latency ↑ + errors ↑** → system instability

---

##### 6. Service Health

Panels:

* Service Up

How to interpret:

* `UP` → metrics being scraped successfully
* `DOWN` → service unreachable or stopped

---

#### Typical Debugging Workflow

##### Scenario 1 — Service is Slow

1. Check **Latency Overview**
2. Identify which metric increases:

   * TTFT → prompt/preprocessing issue
   * ITL → generation slowdown
3. Go to **Latency Breakdown**
4. Confirm root cause:

   * queue → overload
   * prefill → prompt size
   * decode → GPU issue

---

##### Scenario 2 — Throughput Drop

1. Check **Request Concurrency**
2. If:

   * running ↓ and waiting ↑ → queue blockage
3. Check **Cache Efficiency**
4. Check **HTTP errors**

---

##### Scenario 3 — Errors Increasing

1. Check **HTTP status panel**
2. Identify:

   * 4xx → client-side
   * 5xx → server-side
3. Correlate with:

   * latency spike
   * queue increase

---

##### Scenario 4 — After Restart

Expected behavior:

* raw counters reset to zero
* dashboard remains functional
* latency / runtime panels continue normally

---

## Notes

* This dashboard is intended for **engineering debugging**, not business reporting
* All panels are based on **recording rules (canonical metrics)** to ensure stability
* Short time windows (e.g. last 6h) are recommended for analysis
* Works correctly even with ephemeral containers (`docker run --rm`)

---

## Alerting

The system includes a set of **Prometheus-based alerts** to detect service issues and performance degradation.

Alerts are designed to be:

* based on **canonical metrics (recording rules)**
* aware of **business hours (weekday-only availability)**
* focused on **actionable signals**, not noise

---

## Alert Categories

### 1. Service Availability

Detects when the vLLM service is down **during expected business hours**.

* Alert: `VLLMServiceDownBusinessHours`
* Severity: `critical`

Key behavior:

* Will **not trigger on weekends**
* Triggers only when the service is expected to be running

---

### 2. API Errors

Detects elevated HTTP error rates for chat completion requests.

* Alert: `VLLMChatErrorRateHigh`
* Severity: `warning`

Triggered when:

* error rate > 5% for sustained period

---

### 3. Latency Degradation

Detects slow responses from the model.

* Alert: `VLLME2ELatencyP95High`
* Severity: `warning`

Indicates:

* degraded user experience
* possible overload or compute bottleneck

---

### 4. Queue / Backlog

Detects request congestion and queue buildup.

* Alerts:

  * `VLLMQueueTimeP95High`
  * `VLLMRequestsWaitingHigh`
* Severity: `warning`

Indicates:

* system is overloaded
* requests are waiting before execution

---

### 5. Cache Efficiency

Detects inefficient cache usage.

* Alert: `VLLMPrefixCacheHitRateLow`
* Severity: `info`

Indicates:

* prefix cache is not effective
* may impact performance

---

### 6. Traffic Anomaly

Detects unexpected drop in traffic during business hours.

* Alert: `VLLMTrafficDropBusinessHours`
* Severity: `info`

Indicates:

* service is up but receiving no traffic
* possible upstream issue

---

## How Alerts Work

Alerts are defined in:

```id="q2y64f"
deploy/prometheus/alerts.yml
```

Loaded via:

```id="d2y47m"
deploy/prometheus/prometheus.yml
```

Prometheus evaluates rules every:

```id="2fq5xx"
30 seconds
```

---

## How to View Alerts

Open Prometheus UI:

```id="2j0b6r"
http://localhost:9090
```

Navigate to:

```id="zmnq5c"
Alerts
```

You will see:

* Pending alerts
* Firing alerts

---

## How to Validate Alerts

### Service Down

```bash id="p3k6t9"
docker stop <vllm-container>
```

* Wait ~5 minutes
* Verify alert fires (weekday only)

---

### Error Rate

Send invalid requests (bad payloads)

Check:

```promql id="7v0u7y"
http:chat_completions:error_rate5m
```

---

### Latency / Queue

Generate load (concurrent requests)

Observe:

```promql id="8pq0z3"
latency:e2e:p95
latency:queue_time:p95
runtime:requests_waiting
```

---

### Traffic Drop

Stop sending requests while service is running

* Wait ~30 minutes
* Verify alert fires during business hours

---

## Notes

* Alerts are **rule-based**, not statistical
* Thresholds are **initial defaults** and should be tuned over time
* Alerts do not yet send notifications (Phase 3B)

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
