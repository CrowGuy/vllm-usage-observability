# Alerting — VLLM Usage Observability

## Overview

This document defines the alerting strategy for the VLLM Usage Observability system.

Alerting transforms metrics into **actionable operational signals**, allowing engineers to detect:

* service outages
* error spikes
* latency degradation
* traffic anomalies

The system is designed to:

* avoid false positives during **scheduled downtime (weekends)**
* rely on **canonical metrics (recording rules)**
* provide **clear diagnostic signals**, not noisy alerts

---

## Design Principles

### 1. Business-Hours Awareness

The system explicitly encodes expected service availability:

* vLLM is **not expected to run on weekends**
* alerts must **not fire during expected downtime**

This is implemented via:

```promql
service:expected_up:business_hours
```

---

### 2. Metrics Over Logs

All alerts are derived from Prometheus metrics:

* no dependency on logs
* no external state required

---

### 3. Canonical Metrics Only

Alerts must use **recording rules**, not raw metrics.

Examples:

* `service:*`
* `latency:*`
* `runtime:*`
* `http:*`

This ensures:

* stability across vLLM versions
* simplified queries
* consistent semantics

---

### 4. Severity Levels

| Level    | Meaning                                      |
| -------- | -------------------------------------------- |
| critical | immediate action required                    |
| warning  | degradation or potential issue               |
| info     | non-critical signal, useful for optimization |

---

## Alert Definitions

---

### 1. Service Down (Business Hours Only)

**Name**

```
VLLMServiceDownBusinessHours
```

**Severity**

* critical

**Condition**

```promql
(service:expected_up:business_hours == 1)
and
(max(service:up:raw) == 0)
```

**Duration**

* 5 minutes

**Meaning**

* Prometheus cannot scrape the vLLM service
* AND the service is expected to be running

**Why business-hours aware?**

* vLLM is intentionally stopped on weekends
* Avoids false alerts during scheduled downtime

---

### 2. High Error Rate

**Name**

```
VLLMChatErrorRateHigh
```

**Severity**

* warning

**Condition**

```promql
http:chat_completions:error_rate5m > 0.05
```

**Duration**

* 10 minutes

**Meaning**

* More than 5% of chat requests are failing

**Possible causes**

* invalid client requests (4xx)
* backend failures (5xx)
* overload or instability

---

### 3. High End-to-End Latency

**Name**

```
VLLME2ELatencyP95High
```

**Severity**

* warning

**Condition**

```promql
avg(latency:e2e:p95) > 20
```

**Duration**

* 10 minutes

**Meaning**

* User-perceived latency is degraded

---

### 4. High Queue Time

**Name**

```
VLLMQueueTimeP95High
```

**Severity**

* warning

**Condition**

```promql
avg(latency:queue_time:p95) > 1
```

**Duration**

* 10 minutes

**Meaning**

* Requests are waiting too long before execution
* Indicates system overload or insufficient capacity

---

### 5. Too Many Waiting Requests

**Name**

```
VLLMRequestsWaitingHigh
```

**Severity**

* warning

**Condition**

```promql
sum(runtime:requests_waiting) > 5
```

**Duration**

* 10 minutes

**Meaning**

* Backlog is building up

---

### 6. Low Prefix Cache Hit Rate

**Name**

```
VLLMPrefixCacheHitRateLow
```

**Severity**

* info

**Condition**

```promql
avg(runtime:prefix_cache_hit_rate5m) < 0.30
and
sum(http:chat_completions:requests:rate5m) > 0.05
```

**Duration**

* 15 minutes

**Meaning**

* Cache is not effective despite traffic
* Potential performance inefficiency

---

### 7. Traffic Drop (Business Hours Only)

**Name**

```
VLLMTrafficDropBusinessHours
```

**Severity**

* info

**Condition**

```promql
(service:expected_up:business_hours == 1)
and
(sum(http:chat_completions:requests:rate5m) < 0.001)
and
(max(service:up:raw) == 1)
```

**Duration**

* 30 minutes

**Meaning**

* Service is up but receiving almost no traffic
* Possible upstream issue or routing problem

---

## How to Validate Alerts

### Service Down

1. Stop vLLM:

```bash
docker stop <vllm-container>
```

2. Wait 5 minutes

3. Check:

* Prometheus → Alerts
* `VLLMServiceDownBusinessHours` should fire (weekday only)

---

### Error Rate

Send invalid requests:

* malformed payload
* invalid parameters

Check:

```promql
http:chat_completions:error_rate5m
```

---

### Latency / Queue

Generate load:

* concurrent requests
* long prompts

Observe:

* `latency:e2e:p95`
* `latency:queue_time:p95`
* `runtime:requests_waiting`

---

### Traffic Drop

Stop traffic generation (while service is running):

* ensure no requests for >30 minutes
* verify alert triggers during business hours

---

## Known Limitations

* Thresholds are initial estimates and may require tuning
* Traffic anomaly detection is simple (not statistical)
* No alert deduplication or routing (Phase 3B)
* Multi-instance awareness not yet implemented (Phase 4)

---

## Future Work

### Phase 3B — Alert Routing

* Alertmanager integration
* Slack / Email notifications
* alert grouping and deduplication

### Phase 4 — Hardening

* per-instance alerting
* per-model alerting
* adaptive thresholds
* anomaly detection

---

## Summary

This alerting system provides:

* business-aware service monitoring
* stable, metrics-based alerting
* clear mapping between alerts and system behavior

It is designed to evolve incrementally while remaining:

* predictable
* debuggable
* production-ready

---
