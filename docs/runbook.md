# Runbook — VLLM Usage Observability

## Overview

This runbook provides operational guidance for responding to alerts in a **multi-instance vLLM environment**.

It helps engineers:

* identify failing instances
* diagnose performance issues
* distinguish between local and global problems
* apply mitigation steps quickly

---

## System Model

The system consists of:

* multiple vLLM instances
* Prometheus (metrics + alerting)
* Grafana dashboards

Key dimensions:

* `instance_name` → node-level debugging
* `model_name` → usage aggregation
* `env` / `region` → deployment scope

---

## General Workflow

### Step 1 — Identify Alert

From Prometheus:

* alert name
* severity
* scope:

  * `instance`
  * `global`

---

### Step 2 — Determine Scope

| Scope    | Meaning                 |
| -------- | ----------------------- |
| instance | one node affected       |
| global   | entire service affected |

---

### Step 3 — Open Dashboard

```text
http://localhost:3000
```

Use:

* **Engineering Dashboard** → debugging
* **Management Dashboard** → overall health

---

### Step 4 — Apply Playbook

Follow the relevant section below.

---

# Alert Playbooks

---

## 1. Instance Down

**Alert**

```
VLLMInstanceDownBusinessHours
```

---

### Check

```promql
service:up:raw:instance
```

---

### Interpretation

* `0` → instance unreachable
* others still `1` → partial failure

---

### Actions

1. Check container:

```bash
docker ps
```

2. Restart instance:

```bash
docker restart <vllm-container>
```

3. Verify endpoint:

```bash
curl http://<host>:8000/metrics
```

---

### If Multiple Instances Down

Escalate as **global issue**.

---

## 2. All Instances Down

**Alert**

```
VLLMAllInstancesDownBusinessHours
```

---

### Check

```promql
service:up:raw:global
```

---

### Possible Causes

* network outage
* Prometheus cannot reach all targets
* global infrastructure issue

---

### Actions

* verify network connectivity
* check firewall / routing
* verify Prometheus target status

---

## 3. High Error Rate

### Instance

```
VLLMChatErrorRateHighInstance
```

### Global

```
VLLMChatErrorRateHighGlobal
```

---

### Check

```promql
http:chat_completions:error_rate5m:instance
```

```promql
http:chat_completions:error_rate5m:global
```

---

### Diagnosis

| Pattern              | Meaning           |
| -------------------- | ----------------- |
| single instance high | node issue        |
| global high          | system-wide issue |

---

### Actions

* inspect request patterns
* check latency correlation
* validate client inputs
* reduce load if needed

---

## 4. High Latency

### Instance

```
VLLME2ELatencyP95HighInstance
```

### Global

```
VLLME2ELatencyP95HighGlobal
```

---

### Check

```promql
latency:e2e:p95:instance
latency:ttft:p95:instance
latency:itl:p95:instance
```

---

### Diagnosis

| Metric  | Meaning                 |
| ------- | ----------------------- |
| TTFT ↑  | prompt processing issue |
| ITL ↑   | decode slowdown         |
| queue ↑ | overload                |

---

### Actions

1. Check queue:

```promql
latency:queue_time:p95:instance
```

2. Check backlog:

```promql
runtime:requests_waiting:instance
```

3. If overloaded:

* reduce traffic
* scale instances

---

## 5. Queue / Backlog Issues

### Alerts

* `VLLMQueueTimeP95HighInstance`
* `VLLMRequestsWaitingHighInstance`

---

### Check

```promql
runtime:requests_waiting:instance
```

---

### Interpretation

* high waiting → system saturated

---

### Actions

* reduce concurrent load
* scale horizontally
* inspect prompt size

---

## 6. Traffic Drop

### Instance

```
VLLMInstanceTrafficDropBusinessHours
```

### Global

```
VLLMGlobalTrafficDropBusinessHours
```

---

### Check

```promql
http:chat_completions:requests:rate5m:instance
```

---

### Diagnosis

| Pattern              | Meaning           |
| -------------------- | ----------------- |
| single instance drop | routing imbalance |
| global drop          | upstream issue    |

---

### Actions

* verify upstream services
* check API gateway / routing
* confirm clients are active

---

## 7. Cache Efficiency Low

### Alerts

* `VLLMPrefixCacheHitRateLowInstance`
* `VLLMPrefixCacheHitRateLowGlobal`

---

### Check

```promql
runtime:prefix_cache_hit_rate5m:instance
```

---

### Interpretation

* low hit rate → cache not effective

---

### Actions

* analyze workload patterns
* not critical (info-level)

---

# Debugging Patterns (Important)

---

## Pattern 1 — Single Instance Degradation

Symptoms:

* one instance slow
* others normal

Action:

* isolate via `instance_name`
* consider restarting node

---

## Pattern 2 — Partial Capacity Loss

Symptoms:

* one instance down
* others overloaded

Action:

* restore failed instance quickly

---

## Pattern 3 — Global Saturation

Symptoms:

* all instances high latency
* queue builds everywhere

Action:

* scale horizontally
* reduce load

---

## Pattern 4 — Upstream Failure

Symptoms:

* traffic drops to zero
* service still up

Action:

* check upstream systems

---

# Useful Queries

---

### Instance Health

```promql
service:up:raw:instance
```

---

### Active Instances

```promql
count(service:up:raw:instance)
```

---

### Traffic

```promql
http:chat_completions:requests:rate5m:instance
```

---

### Errors

```promql
http:chat_completions:error_rate5m:instance
```

---

### Latency

```promql
latency:e2e:p95:instance
```

---

### Queue

```promql
runtime:requests_waiting:instance
```

---

# Operational Guidelines

---

## When Adding New Instances

* verify labels are correct
* ensure instance appears in dashboard
* confirm metrics ingestion

---

## When Scaling

* monitor Prometheus memory
* avoid cardinality explosion
* use aggregation metrics

---

## Alert Tuning

* adjust thresholds based on real usage
* avoid over-sensitive alerts
* prefer stability over sensitivity

---

# Summary

This runbook enables:

* fast incident response
* clear distinction between instance vs global issues
* effective debugging in multi-instance environments

It is designed for:

> **production operations of vLLM at scale**

---
