# Architecture — VLLM Usage Observability

## 1. Overview

This document defines the system architecture for the VLLM Usage Observability Project.

The system provides:

* Accurate daily usage accounting (requests & tokens)
* Service availability monitoring
* Engineering-level observability (latency, queue, cache, etc.)

It is designed to be:

* resilient to counter resets (e.g. container restart)
* aware of scheduled downtime (weekend shutdown)
* independent of raw metric naming variations
* extensible to future use cases (multi-model, billing, per-user tracking)

---

## 2. High-Level Architecture

```
vLLM (/metrics endpoint)
        ↓
Prometheus (scrape + storage)
        ↓
Recording Rules (semantic layer)
        ↓
Grafana (dashboards)
        ↓
(Optional) Alerting
```

---

## 3. Design Layers

The system is structured into five logical layers.

---

### 3.1 Source Layer — vLLM

**Component**

* vLLM inference service

**Responsibility**

* Expose Prometheus-compatible metrics via `/metrics`

**Key Metrics (validated)**

Usage:

* `vllm:request_success_total`
* `vllm:prompt_tokens_total`
* `vllm:generation_tokens_total`

Runtime:

* `vllm:num_requests_running`
* `vllm:num_requests_waiting`
* `vllm:kv_cache_usage_perc`
* `vllm:prefix_cache_queries_total`
* `vllm:prefix_cache_hits_total`

Latency:

* `vllm:time_to_first_token_seconds`
* `vllm:inter_token_latency_seconds`
* `vllm:e2e_request_latency_seconds`
* `vllm:request_queue_time_seconds`
* `vllm:request_inference_time_seconds`

HTTP Layer:

* `http_requests_total`
* `http_request_duration_seconds`

**Important Notes**

* Metrics are **counter-based and cumulative**
* Counters reset on restart
* Some metrics include labels (e.g. `finished_reason`)
* Deprecated metrics exist and must be avoided in new designs

---

### 3.2 Collection Layer — Prometheus

**Component**

* Prometheus server

**Responsibility**

* Scrape `/metrics` endpoint
* Store time-series data
* Provide query engine (PromQL)

**Key Configurations**

* scrape_interval: 15s (recommended)
* retention: 15–30 days
* persistent storage enabled

**Key Behavior**

* Tracks `up{job="vllm"}` for availability
* Handles counter reset safely via PromQL functions

---

### 3.3 Semantic Layer — Recording Rules

**Component**

* Prometheus recording rules

**Responsibility**

* Convert raw metrics into stable, canonical metrics
* Encapsulate business logic
* Hide metric naming differences
* Standardize query complexity

---

### Why This Layer Is Critical

Raw metrics are:

* version-dependent
* label-complex
* not directly usable for business reporting

This layer creates:

```
usage:requests:daily
usage:tokens:daily
service:availability:business_hours
latency:e2e:p95
```

---

### Example Transformations

#### Requests (exclude abort)

```
sum(increase(vllm:request_success_total{finished_reason=~"stop|length"}[1d]))
```

#### Total Tokens

```
increase(vllm:prompt_tokens_total[1d])
+
increase(vllm:generation_tokens_total[1d])
```

---

### Design Rule

> Dashboards and alerts MUST NOT depend on raw metrics directly.

---

### 3.4 Presentation Layer — Grafana

**Component**

* Grafana dashboards

**Responsibility**

* Visualize canonical metrics
* Serve different audiences

---

#### Dashboard Types

### 1. Management Dashboard

Purpose:

* Business / executive visibility

Characteristics:

* Stable metrics
* Aggregated (daily / weekly)
* Minimal technical detail

Panels:

* Requests / Day
* Prompt Tokens / Day
* Generation Tokens / Day
* Total Tokens / Day
* Service Availability
* 7-day trend

---

### 2. Engineering Dashboard

Purpose:

* System debugging and tuning

Characteristics:

* High granularity
* Real-time metrics
* Detailed breakdowns

Panels:

* Running requests
* Waiting requests
* Latency (p95 / p99)
* KV cache usage
* Prefix cache hit rate
* Queue time vs inference time
* Error rate (HTTP 4xx/5xx)
* Service up/down

---

### Design Principle

Management and Engineering dashboards MUST remain separate.

---

### 3.5 Operations Layer — Alerting & Runbook

**Components**

* Prometheus alert rules
* Runbooks / documentation

---

#### Alert Types

1. Service Down (Business Hours Only)
2. Error Spike (HTTP 4xx/5xx)
3. Latency Degradation
4. Traffic Anomaly

---

#### Special Handling

* Weekend shutdown must NOT trigger alerts
* Alerts must be based on canonical metrics, not raw ones

---

## 4. Data Flow

### Step-by-Step

1. vLLM generates metrics continuously
2. Prometheus scrapes `/metrics`
3. Raw metrics are stored as time-series
4. Recording rules compute canonical metrics
5. Grafana queries canonical metrics
6. Alerts are evaluated on canonical metrics

---

## 5. Core Design Decisions

---

### 5.1 Metrics Over Logs

* Logs are unreliable for accounting
* Metrics are the single source of truth

---

### 5.2 Counter-Based Accounting

All usage metrics must use:

```
increase(...)
```

Reason:

* Handles restart/reset safely
* Prevents overcounting or discontinuity

---

### 5.3 Canonical Metric Layer

* Shields system from metric name changes
* Reduces query duplication
* Improves maintainability

---

### 5.4 Separation of Concerns

| Layer               | Responsibility     |
| ------------------- | ------------------ |
| Usage Metrics       | business reporting |
| Service Metrics     | availability       |
| Engineering Metrics | debugging          |

---

### 5.5 Scheduled Downtime Awareness

* Service is OFF during weekends by design
* Observability system must encode this behavior
* Alerts must respect business schedule

---

## 6. Known Constraints

* vLLM counters reset on restart
* Metric names may vary across versions
* Some metrics are deprecated
* HTTP metrics and vLLM metrics are not identical
* No log-based accounting

---

## 7. Future Extensions

The architecture is designed to support:

* multi-model observability
* per-user usage tracking
* billing integration
* long-term analytics
* capacity planning

---

## 8. Summary

This system is not just a monitoring stack.

It is a **semantic observability system** that:

* transforms raw vLLM metrics into reliable business signals
* separates usage accounting from system health
* ensures correctness under real-world conditions (restart, downtime)
* provides clear visibility for both management and engineering

---
