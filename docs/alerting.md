# Alerting

## Purpose

This document describes the alerting design for the semantic observability layer.

The alerting contract is intentionally simple:

- alerts consume canonical metrics only
- alerts are split by scope
- alerts reflect business-hours-aware availability semantics

## Design Principles

### Canonical Metrics Only

Alerts should consume recording-rule outputs, not raw exporter metrics.

Representative alert inputs:

- `service:*`
- `latency:*`
- `runtime:*`
- `http:*`

### Scope Is Explicit

Alerts are divided into:

- `instance` scope
- `global` scope

This split is part of the design, not just naming.

### Business-Hours Awareness

Availability and traffic-drop alerts respect:

```promql
service:expected_up:business_hours
```

This prevents planned weekend downtime from being treated as an outage.

## Implemented Alerts

### Availability

- `VLLMInstanceDownBusinessHours`
- `VLLMAllInstancesDownBusinessHours`

### API Error Rate

- `VLLMChatErrorRateHighInstance`
- `VLLMChatErrorRateHighGlobal`

### Latency

- `VLLME2ELatencyP95HighInstance`
- `VLLME2ELatencyP95HighGlobal`
- `VLLMQueueTimeP95HighInstance`
- `VLLMQueueTimeP95HighGlobal`

### Runtime / Backlog

- `VLLMRequestsWaitingHighInstance`
- `VLLMRequestsWaitingHighGlobal`

### Cache Efficiency

- `VLLMPrefixCacheHitRateLowInstance`
- `VLLMPrefixCacheHitRateLowGlobal`

### Traffic Drop During Business Hours

- `VLLMInstanceTrafficDropBusinessHours`
- `VLLMGlobalTrafficDropBusinessHours`

## Severity Model

- `critical`: immediate availability concern
- `warning`: degraded user-facing or operational state
- `info`: non-critical but useful optimization or anomaly signal

## Operational Interpretation

### Instance Alerts

Use instance alerts to answer:

- which node is unhealthy?
- is the problem isolated?
- should I open the Engineering dashboard first?

### Global Alerts

Use global alerts to answer:

- is the service broadly unhealthy?
- is the issue systemic rather than local?
- should I start with service-wide health and routing assumptions?

## Important Notes

1. Weekend downtime is expected by design.
2. Traffic-drop alerts assume the service is expected to be running.
3. Error-rate alerts operate at the HTTP layer, not the same semantic layer as vLLM success counters.
4. Queue and backlog alerts are capacity signals, not necessarily hard outages.

## Related Documents

- `docs/runbook.md`
- `docs/semantic-contract.md`
- `docs/adr/004-business-hours-availability.md`
