# Runbook — VLLM Usage Observability

## Overview

This runbook provides step-by-step guidance for responding to alerts in the VLLM observability system.

When an alert fires:

1. Identify the alert type
2. Open the Engineering Dashboard
3. Use PromQL queries to confirm the issue
4. Apply initial mitigation steps

---

## General Workflow

### Step 1 — Identify Alert

From Prometheus Alerts page:

* alert name
* severity
* duration

---

### Step 2 — Open Dashboard

Open Grafana:

```text
http://localhost:3000
```

Go to:

```text
VLLM Usage - Engineering
```

---

### Step 3 — Correlate Metrics

Check:

* concurrency (running / waiting)
* latency (e2e / ttft / itl)
* queue time
* error rate
* cache efficiency

---

### Step 4 — Narrow Down Root Cause

Use the sections below per alert type.

---

## Alert Playbooks

---

### 1. VLLMServiceDownBusinessHours

**Symptom**

* service unreachable

**Check**

```promql
up{job="vllm"}
```

Expected:

* 0

---

**Possible Causes**

* container stopped
* network issue
* port not exposed
* firewall blocking

---

**Actions**

1. Check container:

```bash
docker ps
```

2. Restart:

```bash
docker restart <vllm-container>
```

3. Check metrics endpoint:

```bash
curl http://<vllm-host>:8000/metrics
```

---

### 2. VLLMChatErrorRateHigh

**Symptom**

* high 4xx/5xx responses

---

**Check**

```promql
http:chat_completions:error_rate5m
```

```promql
http:chat_completions:requests:rate5m
```

---

**Possible Causes**

* invalid client requests (4xx)
* backend failures (5xx)
* overload

---

**Actions**

1. Identify status codes (Grafana panel)
2. Check if latency also increased
3. If overload:

   * reduce traffic
   * scale GPU / instances (future Phase 4)

---

### 3. VLLME2ELatencyP95High

**Symptom**

* slow responses

---

**Check**

```promql
latency:e2e:p95
latency:ttft:p95
latency:itl:p95
```

---

**Diagnosis**

| Pattern | Meaning              |
| ------- | -------------------- |
| TTFT ↑  | prompt/prefill issue |
| ITL ↑   | decode slowdown      |
| both ↑  | overall system load  |

---

**Actions**

1. Check queue:

```promql
latency:queue_time:p95
```

2. Check concurrency:

```promql
runtime:requests_waiting
```

3. If queue high:

   * system overloaded
   * reduce traffic or scale

---

### 4. VLLMQueueTimeP95High / VLLMRequestsWaitingHigh

**Symptom**

* requests waiting

---

**Check**

```promql
runtime:requests_waiting
latency:queue_time:p95
```

---

**Possible Causes**

* traffic spike
* insufficient GPU capacity
* long prompts

---

**Actions**

* reduce concurrent load
* analyze prompt sizes
* scale resources (Phase 4)

---

### 5. VLLMPrefixCacheHitRateLow

**Symptom**

* low cache efficiency

---

**Check**

```promql
runtime:prefix_cache_hit_rate5m
```

---

**Possible Causes**

* prompts not reusable
* cache eviction too aggressive

---

**Actions**

* investigate workload pattern
* not urgent (info level)

---

### 6. VLLMTrafficDropBusinessHours

**Symptom**

* service up but no traffic

---

**Check**

```promql
http:chat_completions:requests:rate5m
```

---

**Possible Causes**

* upstream service down
* routing issue
* client stopped sending requests

---

**Actions**

* check upstream systems
* verify API gateway / routing
* confirm clients are active

---

## Useful PromQL Queries

### Availability

```promql
up{job="vllm"}
service:up:raw
```

### Traffic

```promql
http:chat_completions:requests:rate5m
```

### Errors

```promql
http:chat_completions:error_rate5m
```

### Latency

```promql
latency:e2e:p95
latency:queue_time:p95
```

### Runtime

```promql
runtime:requests_running
runtime:requests_waiting
```

---

## Notes

* Alerts are evaluated every 30s
* Most alerts use a `for` window (5–30 minutes)
* Short spikes may not trigger alerts
* System is designed to avoid weekend false positives

---

## Summary

This runbook provides:

* clear debugging steps
* mapping from alert → metrics → action
* fast triage for common failure modes

---
