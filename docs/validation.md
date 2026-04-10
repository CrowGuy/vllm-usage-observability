# Validation Model

## Purpose

Validation in this repository is not limited to "does Prometheus scrape the target?" It verifies the semantic pipeline end to end.

## Validation Layers

### 1. Input Validation

Confirm the expected raw metrics exist:

```promql
vllm:request_success_total
vllm:prompt_tokens_total
vllm:generation_tokens_total
vllm:num_requests_running
vllm:num_requests_waiting
```

For HTTP-level validation:

```promql
http_requests_total
http_request_duration_highr_seconds_bucket
```

### 2. Scrape Validation

Confirm Prometheus can reach the service:

```promql
up{job="vllm"}
```

Expected result:

- `1` for healthy targets

### 3. Semantic Rule Validation

Confirm canonical outputs exist and update:

```promql
usage:tokens:increase1h:global
runtime:requests_waiting:instance
latency:e2e:p95:global
service:up:raw:instance
http:chat_completions:error_rate5m:global
```

### 4. Behavior Validation

Confirm known semantics are preserved:

- request accounting excludes `finished_reason="abort"`
- usage metrics remain correct across counter resets
- rolling 24-hour metrics need warm-up time
- weekend downtime does not page availability alerts

## Known Behaviors

### Counter Reset Behavior

Usage accounting depends on counters queried through `increase()` and `rate()`.

Expected behavior after restart:

- raw counters reset
- time series may fragment
- canonical usage totals remain correct over the selected window

### Rolling 24h Warm-Up

Expected behavior after Prometheus startup:

- `rolling24h` metrics may be `0` or incomplete initially
- full values require enough historical samples to accumulate

### Weekend Shutdown Semantics

Expected behavior on weekends:

- `service:expected_up:business_hours == 0`
- business-hours-aware availability alerts do not fire

### Ephemeral Container Behavior

When vLLM is run with `docker run --rm` or similarly ephemeral lifecycles:

- restarts cause counter resets
- short-lived instances can fragment series
- accounting remains correct only if label identity remains stable enough for aggregation meaning to hold

## Validation Discipline

When diagnosing issues:

1. start with raw metric existence
2. confirm scrape health
3. confirm canonical outputs
4. confirm dashboards and alerts are using the intended canonical metrics

This order avoids confusing input problems with semantic-layer problems.
