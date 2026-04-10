# Runbook

## Purpose

This runbook explains how to respond to alerts using the semantic model defined by the repository.

The first principle is:

- use canonical metrics to determine scope
- then use dashboards to diagnose cause

## Scope Model

Alerts in this repository are intentionally split by scope:

- `instance`: one node or target is unhealthy
- `global`: the service as a whole is unhealthy

That scope is not cosmetic. It should guide diagnosis immediately.

## General Workflow

1. Identify the alert name and scope.
2. Confirm the corresponding canonical metric.
3. Open the appropriate dashboard.
4. Decide whether the problem is local or systemic.
5. Apply the smallest safe remediation.

## Dashboard Choice

Use:

- Management dashboard for service-wide state
- Engineering dashboard for instance-level diagnosis

## Alert Playbooks

### `VLLMInstanceDownBusinessHours`

Check:

```promql
service:up:raw:instance
```

Interpretation:

- one instance at `0` with others healthy indicates a local failure

Actions:

- confirm the container or host is running
- confirm `/metrics` is reachable from Prometheus
- restart the affected instance if appropriate

### `VLLMAllInstancesDownBusinessHours`

Check:

```promql
service:up:raw:global
```

Interpretation:

- global `0` during business hours indicates service-wide unavailability

Actions:

- verify Prometheus target status
- verify network connectivity
- check whether there is a broader infrastructure issue

### `VLLMChatErrorRateHighInstance` and `VLLMChatErrorRateHighGlobal`

Check:

```promql
http:chat_completions:error_rate5m:instance
http:chat_completions:error_rate5m:global
```

Interpretation:

- instance-only increase suggests a local node problem
- global increase suggests system-wide request or service failure

Actions:

- inspect request patterns
- compare with latency and backlog
- validate whether failures are client-side, backend-side, or overload-related

### `VLLME2ELatencyP95HighInstance` and `VLLME2ELatencyP95HighGlobal`

Check:

```promql
latency:e2e:p95:instance
latency:ttft:p95:instance
latency:itl:p95:instance
latency:queue_time:p95:instance
runtime:requests_waiting:instance
```

Interpretation:

- TTFT increase suggests prompt or prefill-side problems
- ITL increase suggests decode-side slowdown
- queue increase suggests overload or insufficient capacity

Actions:

- determine whether the issue is localized or global
- inspect backlog and concurrency
- reduce load or restore capacity as needed

### `VLLMQueueTimeP95HighInstance` and `VLLMQueueTimeP95HighGlobal`

Check:

```promql
latency:queue_time:p95:instance
runtime:requests_waiting:instance
runtime:requests_running:instance
```

Interpretation:

- queue time increase with waiting backlog usually indicates overload

Actions:

- inspect affected instances
- compare waiting vs running requests
- determine whether scaling or traffic reduction is needed

### `VLLMRequestsWaitingHighInstance` and `VLLMRequestsWaitingHighGlobal`

Check:

```promql
runtime:requests_waiting:instance
runtime:requests_waiting:global
```

Interpretation:

- waiting backlog is a direct saturation signal

Actions:

- correlate with queue time and latency
- check whether capacity loss or traffic increase explains the backlog

### `VLLMPrefixCacheHitRateLowInstance` and `VLLMPrefixCacheHitRateLowGlobal`

Check:

```promql
runtime:prefix_cache_hit_rate5m:instance
runtime:prefix_cache_hit_rate5m:global
http:chat_completions:requests:rate5m:instance
http:chat_completions:requests:rate5m:global
```

Interpretation:

- low hit rate under meaningful traffic is an efficiency signal, not necessarily an outage

Actions:

- confirm traffic exists
- inspect whether workload shape changed
- treat as optimization or workload-pattern investigation unless accompanied by user-facing degradation

### `VLLMInstanceTrafficDropBusinessHours` and `VLLMGlobalTrafficDropBusinessHours`

Check:

```promql
http:chat_completions:requests:rate5m:instance
http:chat_completions:requests:rate5m:global
service:up:raw:instance
service:up:raw:global
```

Interpretation:

- service can be healthy while traffic is unexpectedly low

Actions:

- confirm business-hours expectation applies
- determine whether the drop is upstream demand or internal routing failure

## Important Semantics

### Weekends

Availability alerts are business-hours aware. Weekend shutdowns are expected and should not be handled as outages by default.

### Counter Resets

Counter resets usually indicate restart behavior, not accounting corruption. Use canonical usage metrics to confirm business totals before escalating data-integrity concerns.
