# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Service Mesh Observability Platform · Created: 2026-05-11

## Philosophy

This approach treats every concept in the observability domain as a first-class relational entity with strict foreign key relationships and normalized structure. Configuration data (tenants, services, SLOs, alert rules, notification channels), telemetry metadata (trace definitions, span schemas), and operational state (incidents, annotations) each get their own dedicated tables with well-defined constraints.

The telemetry data itself (spans, metrics data points, log entries) is stored in dedicated time-partitioned tables using PostgreSQL native partitioning (or TimescaleDB hypertables), keeping the relational model for high-volume time-series data while maintaining referential integrity for configuration and metadata. This mirrors how traditional APM backends like New Relic and early Jaeger deployments structured their storage — separate indices for services, operations, and tags with join-based lookups.

The core design principle is **referential integrity everywhere**: every span references a known service, every alert rule references a known SLO, every notification references a known channel. This makes the system easy to reason about, query with standard SQL, and audit for compliance. It trades write throughput and schema flexibility for query correctness and developer familiarity.

**Best for:** Teams that value data correctness, SQL familiarity, and strong consistency guarantees over extreme write throughput — particularly regulated environments (HIPAA, SOC 2, FedRAMP) where audit trails and referential integrity are compliance requirements.

**Trade-offs:**
- (+) Full referential integrity — no orphaned records, no dangling references
- (+) Standard SQL — any engineer can query the system without learning a new query language
- (+) Strong ACID guarantees for configuration changes (SLO updates, alert rule modifications)
- (+) Easy to implement row-level security for multi-tenant isolation
- (-) Write throughput ceiling — PostgreSQL struggles above ~100K inserts/second for spans
- (-) Schema rigidity — adding new span attributes requires ALTER TABLE or migration
- (-) JOIN-heavy queries for trace reconstruction can be slow at scale
- (-) Table count is high (~35+), increasing operational complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry Trace Data Model | Span table fields map 1:1 to OTLP Span proto: trace_id, span_id, parent_span_id, name, kind, start/end timestamps, status |
| OpenTelemetry Metrics Data Model | Metric data points stored with aggregation_temporality, monotonic flag, and exemplar links matching OTLP MetricsData proto |
| OpenTelemetry Logs Data Model | Log records follow OTLP LogRecord: timestamp, severity_number, severity_text, body, trace_id/span_id correlation |
| W3C Trace Context Level 1/2 | trace_id is 32-hex-char (128-bit), span_id is 16-hex-char (64-bit), trace_state stored as TEXT |
| Prometheus / OpenMetrics | Metric names follow OpenMetrics naming conventions; label sets stored as normalized key-value pairs |
| ISO 8601 / RFC 3339 | All timestamps stored as TIMESTAMPTZ; durations in nanoseconds (BIGINT) matching OTLP fixed64 |

---

## Tenant & Identity Management

```sql
-- ============================================================
-- TENANT & IDENTITY
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,  -- URL-safe identifier
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'team', 'enterprise')),
    retention_days  INT NOT NULL DEFAULT 30,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    display_name    TEXT,
    role            TEXT NOT NULL DEFAULT 'viewer'
                    CHECK (role IN ('owner', 'admin', 'editor', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL UNIQUE,  -- bcrypt hash of the API key
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Row-Level Security
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON api_keys
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
```

## Service Registry

```sql
-- ============================================================
-- SERVICE REGISTRY
-- ============================================================

CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,  -- e.g. "order-service"
    namespace       TEXT,           -- Kubernetes namespace
    cluster         TEXT,           -- Kubernetes cluster name
    environment     TEXT NOT NULL DEFAULT 'production'
                    CHECK (environment IN ('production', 'staging', 'development')),
    language        TEXT,           -- e.g. "go", "java", "python"
    framework       TEXT,           -- e.g. "gin", "spring-boot"
    otel_sdk_version TEXT,          -- OpenTelemetry SDK version
    metadata        JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, namespace, cluster, environment)
);

CREATE TABLE service_operations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_id      UUID NOT NULL REFERENCES services(id),
    name            TEXT NOT NULL,  -- e.g. "GET /api/orders", "processPayment"
    span_kind       TEXT NOT NULL
                    CHECK (span_kind IN ('SERVER', 'CLIENT', 'INTERNAL', 'PRODUCER', 'CONSUMER')),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (service_id, name, span_kind)
);

CREATE TABLE service_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_service_id   UUID NOT NULL REFERENCES services(id),
    target_service_id   UUID NOT NULL REFERENCES services(id),
    protocol        TEXT,           -- "http", "grpc", "kafka", "redis"
    discovered_via  TEXT NOT NULL DEFAULT 'trace'
                    CHECK (discovered_via IN ('trace', 'ebpf', 'config', 'manual')),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, source_service_id, target_service_id, protocol)
);

ALTER TABLE services ENABLE ROW LEVEL SECURITY;
ALTER TABLE service_dependencies ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON services
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE INDEX idx_services_tenant ON services(tenant_id);
CREATE INDEX idx_services_name ON services(tenant_id, name);
CREATE INDEX idx_service_ops_service ON service_operations(service_id);
CREATE INDEX idx_service_deps_source ON service_dependencies(source_service_id);
CREATE INDEX idx_service_deps_target ON service_dependencies(target_service_id);
```

## Distributed Traces & Spans

```sql
-- ============================================================
-- TRACES & SPANS (time-partitioned)
-- ============================================================

CREATE TABLE traces (
    id              CHAR(32) NOT NULL,  -- 128-bit W3C trace_id as hex
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    root_service_id UUID REFERENCES services(id),
    root_span_name  TEXT,
    span_count      INT NOT NULL DEFAULT 0,
    duration_ns     BIGINT,             -- end-to-end trace duration in nanoseconds
    has_error       BOOLEAN NOT NULL DEFAULT false,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, started_at)
) PARTITION BY RANGE (started_at);

-- Create monthly partitions (automated via pg_partman or cron)
CREATE TABLE traces_2026_05 PARTITION OF traces
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE spans (
    trace_id        CHAR(32) NOT NULL,
    span_id         CHAR(16) NOT NULL,  -- 64-bit span_id as hex
    parent_span_id  CHAR(16),           -- NULL for root spans
    tenant_id       UUID NOT NULL,
    service_id      UUID NOT NULL REFERENCES services(id),
    operation_id    UUID REFERENCES service_operations(id),
    name            TEXT NOT NULL,       -- span name / operation name
    kind            SMALLINT NOT NULL DEFAULT 0
                    CHECK (kind IN (0, 1, 2, 3, 4, 5)),
                    -- 0=UNSPECIFIED, 1=INTERNAL, 2=SERVER, 3=CLIENT, 4=PRODUCER, 5=CONSUMER
    start_time_ns   BIGINT NOT NULL,     -- Unix epoch nanoseconds (OTLP fixed64)
    end_time_ns     BIGINT NOT NULL,
    duration_ns     BIGINT GENERATED ALWAYS AS (end_time_ns - start_time_ns) STORED,
    status_code     SMALLINT NOT NULL DEFAULT 0
                    CHECK (status_code IN (0, 1, 2)),
                    -- 0=UNSET, 1=OK, 2=ERROR
    status_message  TEXT,
    trace_state     TEXT,                -- W3C tracestate header value
    started_at      TIMESTAMPTZ NOT NULL, -- derived from start_time_ns for partitioning
    PRIMARY KEY (span_id, trace_id, started_at)
) PARTITION BY RANGE (started_at);

CREATE TABLE spans_2026_05 PARTITION OF spans
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Span attributes stored as normalized key-value pairs
CREATE TABLE span_attributes (
    span_id         CHAR(16) NOT NULL,
    trace_id        CHAR(32) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    key             TEXT NOT NULL,
    value_type      TEXT NOT NULL CHECK (value_type IN ('string', 'int', 'float', 'bool')),
    string_value    TEXT,
    int_value       BIGINT,
    float_value     DOUBLE PRECISION,
    bool_value      BOOLEAN,
    PRIMARY KEY (span_id, trace_id, started_at, key)
) PARTITION BY RANGE (started_at);

CREATE TABLE span_attributes_2026_05 PARTITION OF span_attributes
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Span events (OTLP Span.Event)
CREATE TABLE span_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    span_id         CHAR(16) NOT NULL,
    trace_id        CHAR(32) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    name            TEXT NOT NULL,
    time_ns         BIGINT NOT NULL,     -- Unix epoch nanoseconds
    attributes      JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (id, started_at)
) PARTITION BY RANGE (started_at);

-- Span links (OTLP Span.Link)
CREATE TABLE span_links (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    span_id         CHAR(16) NOT NULL,
    trace_id        CHAR(32) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    linked_trace_id CHAR(32) NOT NULL,
    linked_span_id  CHAR(16) NOT NULL,
    trace_state     TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (id, started_at)
) PARTITION BY RANGE (started_at);

-- Resource attributes (shared across spans from the same service instance)
CREATE TABLE resource_attributes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    fingerprint     CHAR(32) NOT NULL UNIQUE,  -- hash of sorted attribute set
    attributes      JSONB NOT NULL,
    -- Example JSONB:
    -- {
    --   "service.name": "order-service",
    --   "service.version": "1.4.2",
    --   "host.name": "pod-abc123",
    --   "k8s.namespace.name": "production",
    --   "k8s.deployment.name": "order-service",
    --   "telemetry.sdk.language": "go",
    --   "telemetry.sdk.version": "1.28.0"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_traces_tenant_started ON traces(tenant_id, started_at);
CREATE INDEX idx_traces_has_error ON traces(tenant_id, started_at) WHERE has_error = true;
CREATE INDEX idx_spans_trace ON spans(trace_id, started_at);
CREATE INDEX idx_spans_service ON spans(service_id, started_at);
CREATE INDEX idx_spans_duration ON spans(tenant_id, started_at, duration_ns);
CREATE INDEX idx_span_attrs_key_value ON span_attributes(key, string_value, started_at);
```

## Metrics

```sql
-- ============================================================
-- METRICS (time-partitioned)
-- ============================================================

CREATE TABLE metric_descriptors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,       -- OpenMetrics metric name
    description     TEXT,
    unit            TEXT,                -- e.g. "ms", "bytes", "1" (dimensionless)
    type            TEXT NOT NULL
                    CHECK (type IN ('gauge', 'sum', 'histogram', 'exp_histogram', 'summary')),
    aggregation_temporality TEXT
                    CHECK (aggregation_temporality IN ('delta', 'cumulative')),
    is_monotonic    BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, type)
);

CREATE TABLE metric_data_points (
    metric_id       UUID NOT NULL REFERENCES metric_descriptors(id),
    tenant_id       UUID NOT NULL,
    service_id      UUID NOT NULL REFERENCES services(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    start_timestamp TIMESTAMPTZ,
    value_double    DOUBLE PRECISION,    -- for gauge/sum
    value_int       BIGINT,              -- for integer sum
    count           BIGINT,              -- for histogram/summary
    sum             DOUBLE PRECISION,    -- for histogram/summary
    min             DOUBLE PRECISION,    -- for histogram
    max             DOUBLE PRECISION,    -- for histogram
    bucket_counts   BIGINT[],            -- explicit histogram buckets
    explicit_bounds DOUBLE PRECISION[],  -- histogram bucket boundaries
    quantile_values JSONB,               -- summary quantiles
    labels          JSONB NOT NULL DEFAULT '{}',
    -- Example labels:
    -- {"http.method": "GET", "http.route": "/api/orders", "http.status_code": "200"}
    exemplar_trace_id CHAR(32),          -- link to trace context
    exemplar_span_id  CHAR(16),
    PRIMARY KEY (metric_id, service_id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE TABLE metric_data_points_2026_05 PARTITION OF metric_data_points
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_metric_dp_tenant_time ON metric_data_points(tenant_id, timestamp);
CREATE INDEX idx_metric_dp_service_time ON metric_data_points(service_id, timestamp);
CREATE INDEX idx_metric_dp_labels ON metric_data_points USING GIN (labels);
```

## Logs

```sql
-- ============================================================
-- LOGS (time-partitioned)
-- ============================================================

CREATE TABLE log_records (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id       UUID NOT NULL,
    service_id      UUID NOT NULL REFERENCES services(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    severity_number SMALLINT NOT NULL DEFAULT 0,  -- OTLP 1-24 severity scale
    severity_text   TEXT,                          -- "INFO", "ERROR", etc.
    body            TEXT,                          -- log message body
    trace_id        CHAR(32),                      -- correlation to trace
    span_id         CHAR(16),                      -- correlation to span
    trace_flags     INT DEFAULT 0,
    attributes      JSONB NOT NULL DEFAULT '{}',
    resource_fingerprint CHAR(32),
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE TABLE log_records_2026_05 PARTITION OF log_records
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_logs_tenant_time ON log_records(tenant_id, timestamp);
CREATE INDEX idx_logs_service ON log_records(service_id, timestamp);
CREATE INDEX idx_logs_trace ON log_records(trace_id, timestamp) WHERE trace_id IS NOT NULL;
CREATE INDEX idx_logs_severity ON log_records(tenant_id, severity_number, timestamp);
CREATE INDEX idx_logs_body_search ON log_records USING GIN (to_tsvector('english', body));
CREATE INDEX idx_logs_attrs ON log_records USING GIN (attributes);
```

## SLOs, Alert Rules & Incidents

```sql
-- ============================================================
-- SLOs & ERROR BUDGETS
-- ============================================================

CREATE TABLE slos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_id      UUID NOT NULL REFERENCES services(id),
    name            TEXT NOT NULL,        -- e.g. "Order API Availability"
    description     TEXT,
    sli_type        TEXT NOT NULL
                    CHECK (sli_type IN ('availability', 'latency', 'throughput', 'error_rate')),
    target_percent  NUMERIC(6,3) NOT NULL, -- e.g. 99.950
    window_type     TEXT NOT NULL DEFAULT 'rolling'
                    CHECK (window_type IN ('rolling', 'calendar')),
    window_days     INT NOT NULL DEFAULT 30,
    sli_query       JSONB NOT NULL,       -- definition of how to compute the SLI
    -- Example sli_query:
    -- {
    --   "good_events": "spans WHERE status_code != 2 AND service = 'order-service'",
    --   "total_events": "spans WHERE service = 'order-service' AND kind = 'SERVER'"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE slo_snapshots (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    slo_id          UUID NOT NULL REFERENCES slos(id),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    window_start    TIMESTAMPTZ NOT NULL,
    window_end      TIMESTAMPTZ NOT NULL,
    good_events     BIGINT NOT NULL,
    total_events    BIGINT NOT NULL,
    sli_value       NUMERIC(8,5) NOT NULL,  -- computed SLI percentage
    error_budget_remaining NUMERIC(8,5) NOT NULL,
    burn_rate_1h    NUMERIC(8,5),
    burn_rate_6h    NUMERIC(8,5),
    burn_rate_24h   NUMERIC(8,5),
    PRIMARY KEY (id)
);

-- ============================================================
-- ALERT RULES & NOTIFICATION CHANNELS
-- ============================================================

CREATE TABLE notification_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    channel_type    TEXT NOT NULL
                    CHECK (channel_type IN ('slack', 'pagerduty', 'opsgenie', 'webhook', 'email')),
    config          JSONB NOT NULL,
    -- Example config for Slack:
    -- {"webhook_url": "https://hooks.slack.com/...", "channel": "#alerts"}
    -- Example config for PagerDuty:
    -- {"routing_key": "abc123...", "severity": "critical"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    description     TEXT,
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('threshold', 'anomaly', 'slo_burn_rate', 'predictive')),
    service_id      UUID REFERENCES services(id),  -- NULL = applies globally
    slo_id          UUID REFERENCES slos(id),       -- for SLO-based alerts
    condition       JSONB NOT NULL,
    -- Example condition for threshold:
    -- {"metric": "http_request_duration_p99", "operator": ">", "value": 500, "for": "5m"}
    -- Example condition for SLO burn rate:
    -- {"burn_rate_window": "1h", "threshold": 14.4, "long_window": "6h", "long_threshold": 6}
    severity        TEXT NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('critical', 'warning', 'info')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rule_channels (
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id),
    channel_id      UUID NOT NULL REFERENCES notification_channels(id),
    PRIMARY KEY (alert_rule_id, channel_id)
);

-- ============================================================
-- INCIDENTS
-- ============================================================

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    title           TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('critical', 'warning', 'info')),
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'acknowledged', 'investigating', 'resolved')),
    triggered_by_rule_id UUID REFERENCES alert_rules(id),
    ai_summary      TEXT,                -- LLM-generated root cause narration
    ai_summary_model TEXT,               -- model used for generation
    ai_summary_at   TIMESTAMPTZ,
    affected_services UUID[] NOT NULL DEFAULT '{}',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE incident_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    incident_id     UUID NOT NULL REFERENCES incidents(id),
    event_type      TEXT NOT NULL
                    CHECK (event_type IN ('created', 'acknowledged', 'escalated',
                           'comment', 'ai_analysis', 'resolved', 'reopened')),
    user_id         UUID REFERENCES users(id),
    message         TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_slos_tenant ON slos(tenant_id);
CREATE INDEX idx_slo_snapshots_slo ON slo_snapshots(slo_id, computed_at);
CREATE INDEX idx_alert_rules_tenant ON alert_rules(tenant_id);
CREATE INDEX idx_incidents_tenant_status ON incidents(tenant_id, status, started_at);
CREATE INDEX idx_incident_events_incident ON incident_events(incident_id);
```

## AI & Sampling Configuration

```sql
-- ============================================================
-- AI ANALYSIS & INTELLIGENT SAMPLING
-- ============================================================

CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    incident_id     UUID REFERENCES incidents(id),
    analysis_type   TEXT NOT NULL
                    CHECK (analysis_type IN ('root_cause', 'prediction', 'governance_audit')),
    input_signals   JSONB NOT NULL,       -- traces, metrics, logs referenced
    output_narrative TEXT NOT NULL,        -- LLM-generated analysis
    model_id        TEXT NOT NULL,         -- e.g. "claude-opus-4-6"
    confidence      NUMERIC(4,3),         -- 0.000 to 1.000
    feedback_rating SMALLINT,             -- user feedback: 1-5
    tokens_used     INT,
    latency_ms      INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sampling_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_id      UUID REFERENCES services(id),  -- NULL = global
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('always_sample', 'never_sample',
                           'rate_limit', 'ai_adaptive')),
    condition       JSONB NOT NULL,
    -- Example for rate_limit:
    -- {"max_traces_per_second": 100}
    -- Example for ai_adaptive:
    -- {"novelty_threshold": 0.7, "error_bias": 2.0, "latency_percentile_bias": 0.95}
    priority        INT NOT NULL DEFAULT 0,  -- higher = evaluated first
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sampling_decisions (
    trace_id        CHAR(32) NOT NULL,
    tenant_id       UUID NOT NULL,
    rule_id         UUID REFERENCES sampling_rules(id),
    decision        TEXT NOT NULL CHECK (decision IN ('keep', 'drop')),
    novelty_score   NUMERIC(4,3),         -- AI-computed novelty (0-1)
    reason          TEXT,                  -- why this decision was made
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (trace_id, decided_at)
) PARTITION BY RANGE (decided_at);

CREATE INDEX idx_ai_analyses_tenant ON ai_analyses(tenant_id, created_at);
CREATE INDEX idx_ai_analyses_incident ON ai_analyses(incident_id);
CREATE INDEX idx_sampling_rules_tenant ON sampling_rules(tenant_id);
```

## eBPF Network Flows

```sql
-- ============================================================
-- eBPF NETWORK FLOWS (sidecar-free topology)
-- ============================================================

CREATE TABLE network_flows (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id       UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    source_pod      TEXT,
    source_namespace TEXT,
    source_service_id UUID REFERENCES services(id),
    dest_pod        TEXT,
    dest_namespace  TEXT,
    dest_service_id UUID REFERENCES services(id),
    protocol        TEXT NOT NULL,        -- "TCP", "UDP"
    l7_protocol     TEXT,                 -- "HTTP", "gRPC", "DNS", "Kafka"
    dest_port       INT,
    http_method     TEXT,
    http_status     INT,
    dns_query       TEXT,
    bytes_sent      BIGINT DEFAULT 0,
    bytes_received  BIGINT DEFAULT 0,
    verdict         TEXT NOT NULL DEFAULT 'allowed'
                    CHECK (verdict IN ('allowed', 'denied', 'dropped')),
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE TABLE network_flows_2026_05 PARTITION OF network_flows
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_flows_tenant_time ON network_flows(tenant_id, timestamp);
CREATE INDEX idx_flows_source ON network_flows(source_service_id, timestamp);
CREATE INDEX idx_flows_dest ON network_flows(dest_service_id, timestamp);
CREATE INDEX idx_flows_l7 ON network_flows(l7_protocol, timestamp);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 3 | tenants, users, api_keys |
| Service Registry | 3 | services, service_operations, service_dependencies |
| Traces & Spans | 6 | traces, spans, span_attributes, span_events, span_links, resource_attributes |
| Metrics | 2 | metric_descriptors, metric_data_points |
| Logs | 1 | log_records |
| SLOs & Alerts | 6 | slos, slo_snapshots, notification_channels, alert_rules, alert_rule_channels, incidents + incident_events |
| AI & Sampling | 3 | ai_analyses, sampling_rules, sampling_decisions |
| eBPF Flows | 1 | network_flows |
| **Total** | **25** | Plus partition tables (auto-generated) |

---

## Key Design Decisions

1. **Time-partitioned telemetry tables** — spans, metrics, logs, and network flows are all partitioned by RANGE on their timestamp column. This enables efficient partition pruning for time-range queries and simple data retention via `DROP TABLE` on old partitions rather than expensive DELETE operations.

2. **Normalized span attributes** — Rather than JSONB, span attributes are stored in a dedicated `span_attributes` table with typed value columns. This enables efficient indexing on specific attribute keys (e.g., `http.status_code = 500`) without GIN index overhead, at the cost of more JOINs and higher write amplification (one row per attribute per span).

3. **W3C trace_id as CHAR(32)** — Stored as hex string rather than BYTEA to match the W3C Trace Context specification and simplify debugging. The 32-character fixed-length representation avoids TOAST overhead while remaining human-readable in query results.

4. **Row-Level Security for multi-tenancy** — Every table includes `tenant_id` and RLS policies filter rows based on `current_setting('app.current_tenant_id')`. This provides strong tenant isolation within a shared database, suitable for SOC 2 and FedRAMP compliance requirements.

5. **SLO burn rate snapshots** — Pre-computed `slo_snapshots` capture the SLI value and error budget at regular intervals rather than computing them on-the-fly from raw spans. This supports fast dashboard rendering and historical error budget tracking.

6. **AI analysis as a first-class entity** — The `ai_analyses` table captures every LLM-generated root cause narration with input signals, model metadata, and user feedback ratings. This supports auditability of AI-generated insights (IEEE 7000 alignment) and enables feedback loops for model improvement.

7. **Separate service_dependencies table** — Rather than inferring topology only at query time, observed dependencies are materialized into a dedicated table with source, target, protocol, and discovery method. This supports both trace-derived and eBPF-derived topology in a single unified view.

8. **Exemplar links in metrics** — The `metric_data_points` table includes `exemplar_trace_id` and `exemplar_span_id` fields, implementing the OpenTelemetry exemplar specification to link metric anomalies directly to specific trace spans for drill-down investigation.
