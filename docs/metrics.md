# Metrics Definition

## Purpose

This document describes:

- which raw metrics are relied on
- how they map into canonical metrics
- which semantic rules matter for correctness

The key distinction is:

- raw metrics are implementation inputs
- canonical metrics are the supported observability interface

## Validated Raw Inputs

The repository relies on validated raw metrics from a live vLLM service.

### Usage Inputs

```text
vllm:request_success_total
vllm:prompt_tokens_total
vllm:generation_tokens_total
```

Important detail:

- `vllm:request_success_total` includes `finished_reason`
- only `stop` and `length` are treated as successful business requests
- `abort` is excluded from usage accounting

### Runtime Inputs

```text
vllm:num_requests_running
vllm:num_requests_waiting
vllm:kv_cache_usage_perc
vllm:prefix_cache_queries_total
vllm:prefix_cache_hits_total
vllm:num_preemptions_total
```

### Latency Inputs

```text
vllm:time_to_first_token_seconds
vllm:inter_token_latency_seconds
vllm:e2e_request_latency_seconds
vllm:request_queue_time_seconds
vllm:request_inference_time_seconds
vllm:request_prefill_time_seconds
vllm:request_decode_time_seconds
```

### HTTP Inputs

```text
http_requests_total
http_request_duration_seconds
http_request_duration_highr_seconds
```

## Canonical Metric Families

### Usage

Canonical outputs include:

- `usage:requests:*`
- `usage:prompt_tokens:*`
- `usage:generation_tokens:*`
- `usage:tokens:*`

Semantic rules:

- use `increase()` and `rate()` over counters
- exclude `finished_reason="abort"` from request accounting

### Runtime

Canonical outputs include:

- `runtime:requests_running:*`
- `runtime:requests_waiting:*`
- `runtime:kv_cache_usage_perc:*`
- `runtime:prefix_cache_queries:rate5m:instance`
- `runtime:prefix_cache_hits:rate5m:instance`
- `runtime:prefix_cache_hit_rate5m:*`
- `runtime:preemptions:increase1h:*`

### Latency

Canonical outputs include percentile metrics built from histogram buckets:

- `latency:ttft:*`
- `latency:itl:*`
- `latency:e2e:*`
- `latency:queue_time:*`
- `latency:inference_time:*`
- `latency:prefill_time:*`
- `latency:decode_time:*`

### Service

Canonical service outputs include:

- `service:up:raw:*`
- `service:business_day:asia_taipei`
- `service:expected_up:business_hours`
- `service:availability:business_hours:*`

### HTTP

Canonical HTTP outputs include:

- `http:requests:rate5m:*`
- `http:chat_completions:requests:rate5m:*`
- `http:chat_completions:error_rate5m:*`
- `http:chat_completions:latency:p95:*`

## Aggregation Semantics

Where provided, canonical outputs follow these suffixes:

- `:instance` for diagnosis
- `:model` for model-level usage aggregation
- `:global` for service-level monitoring

This suffix meaning is part of the semantic contract.

## Correctness Rules

### Counter Reset Safety

Usage metrics must remain correct across restarts. For that reason:

- raw counters should not be graphed directly for business accounting
- canonical usage queries use `increase()` and `rate()`

### Request Accounting Filter

`vllm:request_success_total` is not directly equivalent to business request count unless `abort` is excluded.

### HTTP and vLLM Metrics Are Different Layers

These are not interchangeable:

- `http_requests_total` reflects API-layer traffic and failures
- `vllm:request_success_total` reflects successfully processed model requests

Both are useful, but they answer different questions.

### Histogram-Based Latency

Latency percentiles are computed from histogram buckets through recording rules. Dashboards and alerts should consume canonical latency outputs rather than rebuild quantile logic repeatedly.

## Deprecated Raw Metrics

Deprecated metrics may still appear in `/metrics`, for example:

- `vllm:gpu_cache_usage_perc`
- `vllm:gpu_prefix_cache_queries_total`
- `vllm:gpu_prefix_cache_hits_total`
- `vllm:time_per_output_token_seconds`

They are not the preferred basis for new canonical rules in this repository.
