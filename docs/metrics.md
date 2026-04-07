# Metrics Definition — Validated Mapping (vLLM)

## 1. Validation Status

The `/metrics` endpoint has been validated against a live vLLM service instance.

The required MVP usage metrics are present and usable for production observability.

---

## 2. Validated Raw Metrics

### 2.1 Usage Metrics

#### Successful Requests

```text
vllm:request_success_total{finished_reason="stop"| "length"| "abort"}
```

Notes:

* This counter includes a `finished_reason` label
* `stop` and `length` should be treated as successful completed requests
* `abort` must be excluded from business usage accounting

---

#### Prompt Tokens

```text
vllm:prompt_tokens_total
```

Definition:

* Total prefill/input tokens processed

---

#### Generation Tokens

```text
vllm:generation_tokens_total
```

Definition:

* Total output/generated tokens processed

---

### 2.2 Runtime / Engineering Metrics

#### Queue / Concurrency

```text
vllm:num_requests_running
vllm:num_requests_waiting
```

#### Cache

```text
vllm:kv_cache_usage_perc
vllm:prefix_cache_queries_total
vllm:prefix_cache_hits_total
```

#### Latency

```text
vllm:time_to_first_token_seconds
vllm:inter_token_latency_seconds
vllm:e2e_request_latency_seconds
vllm:request_queue_time_seconds
vllm:request_inference_time_seconds
vllm:request_prefill_time_seconds
vllm:request_decode_time_seconds
```

---

### 2.3 HTTP Layer Metrics

```text
http_requests_total
http_request_duration_seconds
http_request_duration_highr_seconds
```

Use cases:

* API error-rate tracking
* handler-level traffic validation
* HTTP-level latency comparison with vLLM internal latency

---

## 3. Canonical Metric Mapping

| Canonical Metric                | Raw Metric                         | Rule                              |
| ------------------------------- | ---------------------------------- | --------------------------------- |
| `usage:requests:daily`          | `vllm:request_success_total`       | exclude `finished_reason="abort"` |
| `usage:prompt_tokens:daily`     | `vllm:prompt_tokens_total`         | use `increase()`                  |
| `usage:generation_tokens:daily` | `vllm:generation_tokens_total`     | use `increase()`                  |
| `usage:tokens:daily`            | prompt + generation                | derived metric                    |
| `service:up:raw`                | `up{job="vllm"}`                   | from Prometheus scrape status     |
| `runtime:requests_running`      | `vllm:num_requests_running`        | gauge                             |
| `runtime:requests_waiting`      | `vllm:num_requests_waiting`        | gauge                             |
| `runtime:kv_cache_usage`        | `vllm:kv_cache_usage_perc`         | gauge                             |
| `runtime:prefix_cache_hit_rate` | prefix hits / prefix queries       | derived ratio                     |
| `latency:ttft`                  | `vllm:time_to_first_token_seconds` | histogram-based percentile        |
| `latency:itl`                   | `vllm:inter_token_latency_seconds` | histogram-based percentile        |
| `latency:e2e`                   | `vllm:e2e_request_latency_seconds` | histogram-based percentile        |

---

## 4. Business Semantics

### Requests / Day

Definition:

* Sum of successfully processed requests excluding aborted requests

Recommended logic:

```text
sum(increase(vllm:request_success_total{finished_reason=~"stop|length"}[1d]))
```

---

### Prompt Tokens / Day

```text
sum(increase(vllm:prompt_tokens_total[1d]))
```

---

### Generation Tokens / Day

```text
sum(increase(vllm:generation_tokens_total[1d]))
```

---

### Total Tokens / Day

```text
sum(increase(vllm:prompt_tokens_total[1d]))
+
sum(increase(vllm:generation_tokens_total[1d]))
```

---

## 5. Important Observations

### 5.1 Counter Reset Safety

The usage metrics are counters and must always be queried via `increase()`.

### 5.2 `request_success_total` Needs Label Filtering

This metric includes multiple `finished_reason` values.
Business dashboards must exclude `abort`.

### 5.3 HTTP Metrics and vLLM Metrics Are Not Identical

`http_requests_total` includes API-layer requests and errors.
`vllm:request_success_total` reflects successfully processed model requests.
They must not be treated as interchangeable.

### 5.4 Deprecated Metrics Exist

The endpoint still exposes deprecated metrics such as:

* `vllm:gpu_cache_usage_perc`
* `vllm:gpu_prefix_cache_queries_total`
* `vllm:gpu_prefix_cache_hits_total`
* `vllm:time_per_output_token_seconds`

Preferred non-deprecated replacements should be used in new dashboards and rules.

---
