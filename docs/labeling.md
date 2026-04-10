# Labeling Strategy

## Purpose

Labeling is what makes the aggregation model correct. Without stable labels, multi-instance semantics break down.

## Design Rules

### 1. Labels Represent Stable Dimensions

Labels should encode stable dimensions such as:

- instance identity
- model identity
- environment
- region

### 2. Labels Must Be Low Cardinality

Avoid labels that grow without bound, such as:

- `user_id`
- `request_id`
- `session_id`
- timestamps

These are unsafe for Prometheus and incompatible with the intended semantic model.

### 3. Labels Should Not Mix Concerns

Each label should mean one thing:

- `instance_name`: instance identity
- `model_name`: served model identity
- `env`: deployment environment
- `region`: deployment region

## Standard Labels

### `instance_name`

Status:

- required for instance-level semantics

Properties:

- stable across restarts
- human-readable
- unique within the deployment scope

### `model_name`

Status:

- strongly recommended for model-level usage aggregation

Purpose:

- aggregate usage across instances serving the same model

### `env`

Status:

- required

Purpose:

- prevent accidental cross-environment aggregation

### `region`

Status:

- recommended

Purpose:

- preserve regional scope in global queries

### `service`

Status:

- required in current Prometheus target labeling

Purpose:

- identify the logical service

### `component`

Status:

- recommended

Purpose:

- distinguish inference from monitoring components

## Why Stable Labels Matter

Stable labels are required for:

- `:instance` diagnosis
- `:model` usage reporting
- `:global` service monitoring
- correct alert scope

If instance identity changes on every restart, the system can still preserve some accounting semantics through counters and windows, but diagnosis and continuity become harder.

## Prometheus Example

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
          region: tw
          instance_name: gpt-oss-120b-a
          model_name: gpt-oss-120b
```

## Aggregation Guidance

### Instance

Use when the question is:

- which node is unhealthy?
- which node is slow?
- which node has backlog?

### Model

Use when the question is:

- how much usage did this model receive?
- how do model groups compare?

### Global

Use when the question is:

- is the service healthy overall?
- is there a cross-instance issue?

## Anti-Patterns

Do not use:

- ephemeral IDs as identity labels
- request-specific values as labels
- business logic encoded implicitly in labels

The aggregation model assumes labels are stable dimensions, not event payloads.
