# Labeling Strategy — VLLM Usage Observability

## Overview

This document defines the labeling strategy for the observability system.

A correct labeling strategy is critical for:

* multi-instance support
* accurate aggregation
* reliable alerting
* scalable Prometheus usage

---

## Design Principles

### 1. Labels Represent Dimensions

Each label must represent a **stable, meaningful dimension**:

* identity (instance)
* logical grouping (model)
* environment (env)
* deployment location (region)

---

### 2. Labels Must Be Stable

Labels should **not change frequently**.

Bad examples:

* container_id
* pod_uid
* random hash

Good examples:

* instance_name
* model_name
* env

---

### 3. Low Cardinality

Avoid labels with high or unbounded values:

* user_id ❌
* request_id ❌
* timestamp ❌

Reason:

* Prometheus memory explosion
* slow queries

---

### 4. Separation of Concerns

Do not mix unrelated concepts into one label.

Example:

| Label         | Meaning                      |
| ------------- | ---------------------------- |
| instance_name | physical machine / container |
| model_name    | model served                 |
| env           | environment                  |

---

## Standard Labels

### 1. `instance_name` (Required)

**Definition**

* unique identifier for each vLLM instance

**Examples**

```text
gpt-oss-120b-a
gpt-oss-120b-b
llama3-70b-a
```

**Properties**

* stable across restarts
* human-readable
* unique within environment

---

### 2. `model_name` (Recommended)

**Definition**

* model served by the instance

**Examples**

```text
gpt-oss-120b
llama3-70b
```

**Purpose**

* aggregation by model
* cost / usage breakdown

---

### 3. `env` (Required)

**Definition**

* deployment environment

**Examples**

```text
prod
staging
dev
```

**Purpose**

* isolate environments
* prevent mixing data

---

### 4. `region` (Optional)

**Definition**

* deployment location

**Examples**

```text
tw
us-east
eu-west
```

**Purpose**

* geo-level aggregation
* latency analysis

---

### 5. `service` (Required)

**Definition**

* logical service name

**Example**

```text
vllm
```

---

### 6. `component` (Recommended)

**Definition**

* system component

**Examples**

```text
inference
monitoring
```

---

## Label Usage in Prometheus

Example:

```yaml
scrape_configs:
  - job_name: vllm
    static_configs:
      - targets:
          - 10.0.0.11:8000
        labels:
          service: vllm
          component: inference
          env: prod
          instance_name: gpt-oss-120b-a
          model_name: gpt-oss-120b
```

---

## Aggregation Strategy

### 1. Per Instance

```promql
sum by (instance_name)
```

Use when:

* debugging
* identifying slow node
* detecting single-node failure

---

### 2. Per Model

```promql
sum by (model_name)
```

Use when:

* comparing models
* usage reporting

---

### 3. Global Aggregation

```promql
sum without (instance_name)
```

Use when:

* system-wide health
* management dashboard

---

## Recommended Patterns

### Usage Metrics

```promql
sum by (model_name) (usage:tokens:increase1h)
```

---

### Latency Metrics

```promql
avg by (instance_name) (latency:e2e:p95)
```

---

### Runtime Metrics

```promql
sum by (instance_name) (runtime:requests_waiting)
```

---

## Anti-Patterns

### 1. Mixing Dimensions

Bad:

```text
instance = gpt-oss-120b-a-prod
```

Good:

```text
instance_name = gpt-oss-120b-a
env = prod
```

---

### 2. High Cardinality Labels

Bad:

```text
user_id = 123456
request_id = abcdef
```

---

### 3. Using IP as Identity

Bad:

```text
instance = 10.0.0.11:8000
```

Reason:

* changes over time
* not human-readable

---

## Dashboard Implications

Dashboards should support:

* filtering by `instance_name`
* filtering by `model_name`
* filtering by `env`

Typical Grafana variables:

* instance_name
* model_name

---

## Alerting Implications

### Global Alerts

```promql
sum(...) > threshold
```

---

### Per-Instance Alerts

```promql
by (instance_name)
```

---

## Migration Notes

When moving from single-instance to multi-instance:

1. Add labels in `prometheus.yml`
2. Update recording rules to include aggregation
3. Update dashboards to support filtering
4. Update alerts to distinguish global vs instance-level

---

## Summary

A correct labeling strategy enables:

* scalable observability
* accurate aggregation
* meaningful dashboards
* reliable alerting

The most critical labels are:

* `instance_name`
* `model_name`
* `env`

These form the foundation for all future system extensions.

---
