# vllm-usage-observability

vllm-usage-observability/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── metrics.md
│   ├── runbook.md
│   ├── troubleshooting.md
│   └── decisions/
│       ├── 0001-metrics-over-logs.md
│       ├── 0002-business-hours-semantics.md
│       └── 0003-counter-reset-handling.md
│
├── deploy/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   ├── alerts.yml
│   │   └── recording_rules.yml
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/
│       │   └── dashboards/
│       └── dashboards/
│           ├── management.json
│           └── engineering.json
│
├── config/
│   ├── targets/
│   │   ├── dev.yml
│   │   ├── staging.yml
│   │   └── prod.yml
│   └── metric_mapping.yml
│
├── scripts/
│   ├── validate_metrics.sh
│   ├── smoke_test.sh
│   └── generate_test_load.py
│
└── tests/
    ├── test_metric_presence.md
    ├── test_counter_reset.md
    ├── test_weekend_shutdown.md
    └── test_queries.md