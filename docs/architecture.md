# Architecture — VLLM Usage Observability

## Overview

This system provides a **multi-instance observability platform** for vLLM services, covering:

* usage tracking
* performance diagnostics
* service monitoring
* alerting

It is designed to scale from a single instance to **multiple vLLM deployments** across environments and regions.

---

## High-Level Architecture

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

---

## Multi-Instance Architecture

```text
[vLLM A]   [vLLM B]   [vLLM C]
   ↓           ↓           ↓
        Prometheus
             ↓
   Recording Rules
   (instance / model / global)
             ↓
     Alerts + Grafana
```

---

## Components

---

### 1. vLLM Instances

Each vLLM instance:

* exposes `/metrics`
* runs independently (container / machine)
* serves a specific model

Provides:

* usage counters (requests, tokens)
* latency histograms (TTFT, ITL, E2E)
* runtime metrics (queue, concurrency)
* cache metrics (KV cache, prefix cache)
* HTTP metrics

---

### 2. Prometheus

Responsibilities:

* scrape multiple vLLM targets
* attach stable labels:

  * `instance_name`
  * `env`
  * `region`
* store time series
* evaluate rules

---

### 3. Labeling Layer

Defined in:

```text
docs/labeling.md
```

Key labels:

| Label         | Purpose             |
| ------------- | ------------------- |
| instance_name | identify instance   |
| model_name    | identify model      |
| env           | environment         |
| region        | deployment location |

This layer enables:

* aggregation
* filtering
* alert scoping

---

### 4. Recording Rules (Semantic Layer)

Defined in:

```text
deploy/prometheus/recording_rules.yml
```

Transforms raw metrics into canonical metrics with **three aggregation levels**:

#### 1. Instance Level

```text
*:instance
```

Used for:

* debugging
* per-node visibility

---

#### 2. Model Level

```text
*:model
```

Used for:

* usage reporting
* cost analysis

---

#### 3. Global Level

```text
*:global
```

Used for:

* service-wide monitoring
* management dashboard

---

### 5. Alerting Rules (Evaluation Layer)

Defined in:

```text
deploy/prometheus/alerts.yml
```

Alerts are categorized into:

* instance-level alerts
* global alerts

Examples:

| Scope    | Example                         |
| -------- | ------------------------------- |
| instance | instance down, latency high     |
| global   | traffic drop, global error rate |

---

### 6. Business-Hours Awareness

The system encodes expected service availability:

```promql
service:expected_up:business_hours
```

Used to:

* suppress alerts during weekends
* avoid false positives

---

### 7. Grafana

Provides two types of dashboards:

---

#### Engineering Dashboard

Purpose:

* debugging
* per-instance visibility

Features:

* instance filtering
* latency breakdown
* queue analysis

---

#### Management Dashboard

Purpose:

* business and usage overview

Features:

* usage metrics (requests, tokens)
* model-level aggregation
* service availability

---

## Data Flow

### Step 1 — Metrics Emission

Each vLLM instance exposes:

```text
/metrics
```

---

### Step 2 — Scraping

Prometheus scrapes:

* multiple targets
* attaches labels (`instance_name`, `env`, `region`)

---

### Step 3 — Semantic Transformation

Recording rules produce:

* `usage:*`
* `latency:*`
* `runtime:*`
* `service:*`
* `http:*`

At:

* instance level
* model level
* global level

---

### Step 4 — Alert Evaluation

Prometheus evaluates alert rules:

* business-hours aware
* instance vs global

---

### Step 5 — Visualization

Grafana dashboards query canonical metrics.

---

## Design Principles

---

### 1. Multi-Instance First

The system assumes:

* multiple instances
* horizontal scaling

---

### 2. Separation of Concerns

| Layer            | Responsibility  |
| ---------------- | --------------- |
| raw metrics      | vLLM exporter   |
| semantic metrics | recording rules |
| alert logic      | alert rules     |
| visualization    | Grafana         |

---

### 3. Label-Driven Architecture

All aggregation depends on:

* `instance_name`
* `model_name`
* `env`
* `region`

---

### 4. Metrics-First Design

* no dependency on logs
* fully Prometheus-based

---

### 5. Config-as-Code

All configuration is version-controlled:

* Prometheus config
* recording rules
* alert rules
* dashboards

---

## Failure Model

The system distinguishes:

### 1. Instance Failure

* one instance down
* detected by instance-level alerts

---

### 2. Partial Degradation

* one instance slow
* detected via latency / queue metrics

---

### 3. Global Failure

* all instances down
* or traffic drops to zero

---

## Future Extensions

---

### Phase 3B — Alert Routing

* Alertmanager
* Slack / Email integration

---

### Phase 4 — Hardening (Current Phase)

* multi-instance support
* labeling strategy
* aggregation layers

---

### Phase 5 — Advanced Observability

* anomaly detection
* adaptive thresholds
* cost attribution per model / team

---

## Summary

This architecture provides:

* scalable multi-instance observability
* consistent metric semantics
* clear separation between debugging and reporting
* reliable alerting with business-aware logic

It evolves from a simple monitoring setup into a:

> **production-ready observability platform for vLLM systems**

---
