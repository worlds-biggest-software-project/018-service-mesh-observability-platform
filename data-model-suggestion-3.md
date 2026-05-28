# Data Model Suggestion 3: Hybrid Relational + ClickHouse Columnar

> Project: Service Mesh Observability Platform · Created: 2026-05-11

## Philosophy

This approach uses a dual-database architecture: PostgreSQL for configuration, metadata, and operational state (tenants, services, SLOs, alert rules, incidents) and ClickHouse for high-volume telemetry data (spans, metrics, logs, network flows). This is the architecture that SigNoz, the most successful open-source unified observability platform, uses in production — and for good reason. Each database plays to its strengths: PostgreSQL provides ACID transactions, foreign keys, and row-level security for the data that needs it; ClickHouse provides columnar compression, vectorized query execution, and MergeTree-based aggregation for the data that needs to be scanned at billions of rows per second.

The design is heavily informed by SigNoz's ClickHouse schema (documented at signoz.io) and Grafana Tempo's columnar storage approach, adapted to include the AI-native features (root cause analysis, predictive SLO breach detection, intelligent sampling) that differentiate this platform. ClickHouse tables use the ReplicatedMergeTree engine family with materialized views for pre-aggregated RED metrics, and the `Map` column type for OpenTelemetry's variable-cardinality attributes — avoiding the write amplification that plagues normalized attribute tables in relational designs.

The core design principle is **right tool for the right data**: relational where consistency matters, columnar where scan speed matters. Writes flow through an OTLP-compatible collector into ClickHouse at hundreds of thousands of rows per second, while configuration changes go through a standard REST API into PostgreSQL. The two databases are linked by shared identifiers (tenant_id, service names, trace_id) but do not have cross-database foreign keys — accepting this referential looseness in exchange for independent scaling.

**Best for:** Teams building for production scale from day one who need to handle 100K+ spans/second, want sub-second dashboard queries over billions of rows, and are comfortable operating two database systems. This is the architecture proven by SigNoz, Grafana, and most modern observability backends.

**Trade-offs:**
- (+) ClickHouse handles 100K-1M+ inserts/second for telemetry — 10-100x PostgreSQL
- (+) Columnar compression reduces telemetry storage by 80-95% compared to row-based
- (+) Vectorized query execution enables sub-second aggregation over billions of spans
- (+) Materialized views pre-compute RED metrics with zero query-time cost
- (+) PostgreSQL retains full ACID/FK/RLS for configuration and operational state
- (-) Two databases to operate, backup, monitor, and upgrade
- (-) No cross-database foreign keys — orphaned references are possible
- (-) ClickHouse's eventual consistency model (async replication) can lose recent data on node failure
- (-) ClickHouse mutations (UPDATE/DELETE) are expensive — data corrections require rewriting parts
- (-) Team needs expertise in both PostgreSQL and ClickHouse SQL dialects

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry Trace Data Model | ClickHouse span columns map directly to OTLP Span proto fields; Map columns store variable attributes exactly as OTLP KeyValue arrays |
| OpenTelemetry Metrics Data Model | Metric data points stored with aggregation_temporality and exemplar_trace_id matching OTLP proto |
| OpenTelemetry Logs Data Model | Log table columns follow OTLP LogRecord: severity_number, severity_text, body, trace_id/span_id |
| W3C Trace Context Level 1/2 | trace_id stored as FixedString(32), span_id as String, matching SigNoz convention |
| Prometheus / OpenMetrics | Metric names follow OpenMetrics conventions; label maps use ClickHouse Map type |
| ISO 8601 | PostgreSQL uses TIMESTAMPTZ; ClickHouse uses DateTime64(9) for nanosecond precision |

---

## PostgreSQL: Configuration & Operational State

```sql
-- ============================================================
-- POSTGRESQL: CONFIGURATION & OPERATIONAL STATE
-- ============================================================
-- This database handles all mutable configuration, user management,
-- and operational state that requires ACID guarantees.

-- TENANTS & USERS
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'team', 'enterprise')),
    retention_days  INT NOT NULL DEFAULT 30,
    max_spans_per_sec INT NOT NULL DEFAULT 10000,
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
    UNIQUE (tenant_id, email)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- SERVICE REGISTRY (authoritative metadata lives in PostgreSQL)
CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    namespace       TEXT,
    cluster         TEXT,
    environment     TEXT NOT NULL DEFAULT 'production',
    metadata        JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, namespace, cluster, environment)
);

-- SLOs
CREATE TABLE slos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_name    TEXT NOT NULL,
    name            TEXT NOT NULL,
    sli_type        TEXT NOT NULL
                    CHECK (sli_type IN ('availability', 'latency', 'throughput', 'error_rate')),
    target_percent  NUMERIC(6,3) NOT NULL,
    window_type     TEXT NOT NULL DEFAULT 'rolling',
    window_days     INT NOT NULL DEFAULT 30,
    sli_query       JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ALERT RULES
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('threshold', 'anomaly', 'slo_burn_rate', 'predictive')),
    service_name    TEXT,
    slo_id          UUID REFERENCES slos(id),
    condition       JSONB NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- NOTIFICATION CHANNELS
CREATE TABLE notification_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    channel_type    TEXT NOT NULL
                    CHECK (channel_type IN ('slack', 'pagerduty', 'opsgenie', 'webhook', 'email')),
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rule_channels (
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES notification_channels(id) ON DELETE CASCADE,
    PRIMARY KEY (alert_rule_id, channel_id)
);

-- INCIDENTS
CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    title           TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning',
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'acknowledged', 'investigating', 'resolved')),
    triggered_by_rule_id UUID REFERENCES alert_rules(id),
    ai_summary      TEXT,
    ai_model        TEXT,
    ai_confidence   NUMERIC(4,3),
    affected_services TEXT[] NOT NULL DEFAULT '{}',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE incident_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    incident_id     UUID NOT NULL REFERENCES incidents(id),
    event_type      TEXT NOT NULL,
    user_id         UUID REFERENCES users(id),
    message         TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- SAMPLING RULES
CREATE TABLE sampling_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_name    TEXT,
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('always_sample', 'never_sample',
                           'rate_limit', 'ai_adaptive')),
    condition       JSONB NOT NULL,
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- AI ANALYSES (stored in PostgreSQL for relational linking to incidents)
CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    incident_id     UUID REFERENCES incidents(id),
    analysis_type   TEXT NOT NULL,
    input_signals   JSONB NOT NULL,
    output_narrative TEXT NOT NULL,
    model_id        TEXT NOT NULL,
    confidence      NUMERIC(4,3),
    feedback_rating SMALLINT,
    tokens_used     INT,
    latency_ms      INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Row-Level Security
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE services ENABLE ROW LEVEL SECURITY;
ALTER TABLE slos ENABLE ROW LEVEL SECURITY;
ALTER TABLE alert_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE incidents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON services
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON slos
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON alert_rules
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON incidents
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Indexes
CREATE INDEX idx_services_tenant ON services(tenant_id, name);
CREATE INDEX idx_slos_tenant ON slos(tenant_id, service_name);
CREATE INDEX idx_alerts_tenant ON alert_rules(tenant_id);
CREATE INDEX idx_incidents_tenant_status ON incidents(tenant_id, status, started_at);
CREATE INDEX idx_incident_events_incident ON incident_events(incident_id);
CREATE INDEX idx_ai_analyses_incident ON ai_analyses(incident_id);
```

## ClickHouse: Telemetry Data (Spans)

```sql
-- ============================================================
-- CLICKHOUSE: DISTRIBUTED TRACE SPANS
-- ============================================================
-- Modeled after SigNoz's signoz_index_v3 schema with adaptations
-- for AI-native features (sampling decisions, novelty scores).

-- Resource metadata lookup table (deduplicated)
CREATE TABLE traces_resource
(
    `labels`                    String CODEC(ZSTD(5)),
    `fingerprint`               String CODEC(ZSTD(1)),
    `seen_at_ts_bucket_start`   Int64  CODEC(Delta(8), ZSTD(1))
)
ENGINE = ReplicatedReplacingMergeTree
ORDER BY (fingerprint, seen_at_ts_bucket_start)
TTL toDateTime(seen_at_ts_bucket_start) + toIntervalDay(90)
SETTINGS index_granularity = 8192;

-- Main span index table
CREATE TABLE spans
(
    -- Time bucketing for efficient range queries (1800-second = 30-minute buckets)
    `ts_bucket_start`       UInt64 CODEC(DoubleDelta, LZ4),

    -- Resource fingerprint (join key to traces_resource)
    `resource_fingerprint`  String CODEC(ZSTD(1)),

    -- Core OTLP Span fields (W3C Trace Context)
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `trace_id`              FixedString(32) CODEC(ZSTD(1)),
    `span_id`               String CODEC(ZSTD(1)),
    `parent_span_id`        String CODEC(ZSTD(1)),
    `trace_state`           String CODEC(ZSTD(1)),
    `name`                  LowCardinality(String) CODEC(ZSTD(1)),
    `kind`                  Int8 CODEC(T64, ZSTD(1)),
        -- 0=UNSPECIFIED, 1=INTERNAL, 2=SERVER, 3=CLIENT, 4=PRODUCER, 5=CONSUMER
    `kind_string`           LowCardinality(String) CODEC(ZSTD(1)),
    `duration_nano`         UInt64 CODEC(T64, ZSTD(1)),
    `status_code`           Int16 CODEC(T64, ZSTD(1)),
        -- 0=UNSET, 1=OK, 2=ERROR
    `status_message`        String CODEC(ZSTD(1)),
    `has_error`             Bool CODEC(ZSTD(1)),

    -- Tenant isolation
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),

    -- Commonly queried HTTP/RPC attributes (materialized for performance)
    `http_method`           LowCardinality(String) CODEC(ZSTD(1)),
    `http_url`              String CODEC(ZSTD(2)),
    `http_route`            LowCardinality(String) CODEC(ZSTD(1)),
    `http_host`             LowCardinality(String) CODEC(ZSTD(1)),
    `http_status_code`      Int16 CODEC(T64, ZSTD(1)),
    `rpc_method`            LowCardinality(String) CODEC(ZSTD(1)),
    `rpc_system`            LowCardinality(String) CODEC(ZSTD(1)),
    `response_status_code`  LowCardinality(String) CODEC(ZSTD(1)),

    -- Database attributes
    `db_system`             LowCardinality(String) CODEC(ZSTD(1)),
    `db_name`               LowCardinality(String) CODEC(ZSTD(1)),
    `db_operation`          LowCardinality(String) CODEC(ZSTD(1)),

    -- Messaging attributes
    `messaging_system`      LowCardinality(String) CODEC(ZSTD(1)),
    `messaging_operation`   LowCardinality(String) CODEC(ZSTD(1)),

    -- Variable attributes as typed Maps (OpenTelemetry KeyValue arrays)
    `attributes_string`     Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    `attributes_number`     Map(LowCardinality(String), Float64) CODEC(ZSTD(1)),
    `attributes_bool`       Map(LowCardinality(String), Bool) CODEC(ZSTD(1)),

    -- Resource attributes (denormalized for query convenience)
    `resources_string`      Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Span events (OTLP Span.Event) stored as nested arrays
    `event_names`           Array(String) CODEC(ZSTD(2)),
    `event_times`           Array(DateTime64(9)) CODEC(ZSTD(1)),
    `event_attributes`      Array(Map(LowCardinality(String), String)) CODEC(ZSTD(1)),

    -- Span links (OTLP Span.Link) stored as nested arrays
    `link_trace_ids`        Array(FixedString(32)) CODEC(ZSTD(1)),
    `link_span_ids`         Array(String) CODEC(ZSTD(1)),

    -- AI-native: sampling decision metadata
    `sampling_decision`     LowCardinality(String) CODEC(ZSTD(1)),
        -- 'kept', 'ai_kept', 'ai_dropped' (empty = no decision recorded)
    `novelty_score`         Float32 CODEC(ZSTD(1)),
        -- AI-computed novelty score (0.0 = routine, 1.0 = highly novel)

    -- Scope (instrumentation library)
    `scope_name`            LowCardinality(String) CODEC(ZSTD(1)),
    `scope_version`         LowCardinality(String) CODEC(ZSTD(1))
)
ENGINE = ReplicatedMergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, ts_bucket_start, resource_fingerprint, has_error, name, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(30)  -- configurable per tenant
SETTINGS index_granularity = 8192;

-- Secondary indexes for high-cardinality lookups
ALTER TABLE spans ADD INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE spans ADD INDEX idx_span_id span_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE spans ADD INDEX idx_duration duration_nano TYPE minmax GRANULARITY 1;
ALTER TABLE spans ADD INDEX idx_http_status http_status_code TYPE set(0) GRANULARITY 1;
```

## ClickHouse: Telemetry Data (Metrics)

```sql
-- ============================================================
-- CLICKHOUSE: METRICS
-- ============================================================

CREATE TABLE metric_samples
(
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),
    `metric_name`           LowCardinality(String) CODEC(ZSTD(1)),
    `metric_description`    String CODEC(ZSTD(2)),
    `metric_unit`           LowCardinality(String) CODEC(ZSTD(1)),
    `metric_type`           LowCardinality(String) CODEC(ZSTD(1)),
        -- 'gauge', 'sum', 'histogram', 'exp_histogram', 'summary'
    `aggregation_temporality` LowCardinality(String) CODEC(ZSTD(1)),
        -- 'delta', 'cumulative'
    `is_monotonic`          Bool CODEC(ZSTD(1)),
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `start_timestamp`       DateTime64(9) CODEC(DoubleDelta, LZ4),

    -- Gauge/Sum values
    `value`                 Float64 CODEC(ZSTD(1)),

    -- Histogram fields
    `count`                 UInt64 CODEC(T64, ZSTD(1)),
    `sum`                   Float64 CODEC(ZSTD(1)),
    `min`                   Float64 CODEC(ZSTD(1)),
    `max`                   Float64 CODEC(ZSTD(1)),
    `bucket_counts`         Array(UInt64) CODEC(ZSTD(1)),
    `explicit_bounds`       Array(Float64) CODEC(ZSTD(1)),

    -- Labels (OpenMetrics label set)
    `labels`                Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Resource attributes
    `resource_fingerprint`  String CODEC(ZSTD(1)),
    `resources_string`      Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Exemplar (link to trace)
    `exemplar_trace_id`     FixedString(32) CODEC(ZSTD(1)),
    `exemplar_span_id`      String CODEC(ZSTD(1)),

    -- Scope
    `scope_name`            LowCardinality(String) CODEC(ZSTD(1)),
    `scope_version`         LowCardinality(String) CODEC(ZSTD(1))
)
ENGINE = ReplicatedMergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, metric_name, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(90)
SETTINGS index_granularity = 8192;

ALTER TABLE metric_samples ADD INDEX idx_metric_labels labels TYPE ngrambf_v1(4, 256, 2, 0) GRANULARITY 1;
```

## ClickHouse: Telemetry Data (Logs)

```sql
-- ============================================================
-- CLICKHOUSE: LOG RECORDS
-- ============================================================
-- Follows the OpenTelemetry Logs Data Model specification.

CREATE TABLE log_records
(
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `observed_timestamp`    DateTime64(9) CODEC(DoubleDelta, LZ4),
    `id`                    String CODEC(ZSTD(1)),
    `trace_id`              FixedString(32) CODEC(ZSTD(1)),
    `span_id`               String CODEC(ZSTD(1)),
    `trace_flags`           UInt32 DEFAULT 0,
    `severity_text`         LowCardinality(String) CODEC(ZSTD(1)),
    `severity_number`       UInt8 CODEC(ZSTD(1)),
    `body`                  String CODEC(ZSTD(2)),

    -- Typed attribute maps
    `attributes_string`     Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    `attributes_number`     Map(LowCardinality(String), Float64) CODEC(ZSTD(1)),
    `attributes_bool`       Map(LowCardinality(String), Bool) CODEC(ZSTD(1)),

    -- Resource attributes
    `resource_fingerprint`  String CODEC(ZSTD(1)),
    `resources_string`      Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Scope
    `scope_name`            LowCardinality(String) CODEC(ZSTD(1)),
    `scope_version`         LowCardinality(String) CODEC(ZSTD(1))
)
ENGINE = ReplicatedMergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, severity_number, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(30)
SETTINGS index_granularity = 8192;

ALTER TABLE log_records ADD INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE log_records ADD INDEX idx_body body TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 1;
```

## ClickHouse: eBPF Network Flows

```sql
-- ============================================================
-- CLICKHOUSE: eBPF NETWORK FLOWS
-- ============================================================

CREATE TABLE network_flows
(
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `source_pod`            LowCardinality(String) CODEC(ZSTD(1)),
    `source_namespace`      LowCardinality(String) CODEC(ZSTD(1)),
    `source_service`        LowCardinality(String) CODEC(ZSTD(1)),
    `dest_pod`              LowCardinality(String) CODEC(ZSTD(1)),
    `dest_namespace`        LowCardinality(String) CODEC(ZSTD(1)),
    `dest_service`          LowCardinality(String) CODEC(ZSTD(1)),
    `protocol`              LowCardinality(String) CODEC(ZSTD(1)),
    `l7_protocol`           LowCardinality(String) CODEC(ZSTD(1)),
    `dest_port`             UInt16 CODEC(T64, ZSTD(1)),
    `http_method`           LowCardinality(String) CODEC(ZSTD(1)),
    `http_status`           UInt16 CODEC(T64, ZSTD(1)),
    `dns_query`             String CODEC(ZSTD(2)),
    `bytes_sent`            UInt64 CODEC(T64, ZSTD(1)),
    `bytes_received`        UInt64 CODEC(T64, ZSTD(1)),
    `verdict`               LowCardinality(String) CODEC(ZSTD(1))
        -- 'allowed', 'denied', 'dropped'
)
ENGINE = ReplicatedMergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, source_service, dest_service, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(14)
SETTINGS index_granularity = 8192;
```

## ClickHouse: Materialized Views for RED Metrics

```sql
-- ============================================================
-- CLICKHOUSE: MATERIALIZED VIEWS — PRE-AGGREGATED RED METRICS
-- ============================================================
-- These materialized views trigger on INSERT to the spans table
-- and incrementally maintain per-service, per-operation RED metrics.

-- Target table for 1-minute RED aggregates
CREATE TABLE red_metrics_1m
(
    `tenant_id`         LowCardinality(String),
    `service_name`      LowCardinality(String),
    `operation_name`    LowCardinality(String),
    `span_kind`         LowCardinality(String),
    `bucket_start`      DateTime CODEC(DoubleDelta, LZ4),
    `request_count`     AggregateFunction(count, UInt64),
    `error_count`       AggregateFunction(countIf, Bool),
    `duration_sum`      AggregateFunction(sum, UInt64),
    `duration_min`      AggregateFunction(min, UInt64),
    `duration_max`      AggregateFunction(max, UInt64),
    `duration_quantiles` AggregateFunction(quantiles(0.5, 0.75, 0.9, 0.95, 0.99), UInt64)
)
ENGINE = ReplicatedAggregatingMergeTree
PARTITION BY toYYYYMM(bucket_start)
ORDER BY (tenant_id, service_name, operation_name, span_kind, bucket_start)
TTL bucket_start + toIntervalDay(90)
SETTINGS index_granularity = 8192;

-- Materialized view that populates red_metrics_1m on every span INSERT
CREATE MATERIALIZED VIEW red_metrics_1m_mv TO red_metrics_1m AS
SELECT
    tenant_id,
    resources_string['service.name'] AS service_name,
    name AS operation_name,
    kind_string AS span_kind,
    toStartOfMinute(timestamp) AS bucket_start,
    countState() AS request_count,
    countIfState(has_error) AS error_count,
    sumState(duration_nano) AS duration_sum,
    minState(duration_nano) AS duration_min,
    maxState(duration_nano) AS duration_max,
    quantilesState(0.5, 0.75, 0.9, 0.95, 0.99)(duration_nano) AS duration_quantiles
FROM spans
WHERE kind IN (2, 4)  -- SERVER and PRODUCER spans only (entry points)
GROUP BY
    tenant_id, service_name, operation_name, span_kind, bucket_start;

-- Target table for 1-hour RED aggregates
CREATE TABLE red_metrics_1h
(
    `tenant_id`         LowCardinality(String),
    `service_name`      LowCardinality(String),
    `operation_name`    LowCardinality(String),
    `bucket_start`      DateTime CODEC(DoubleDelta, LZ4),
    `request_count`     AggregateFunction(count, UInt64),
    `error_count`       AggregateFunction(countIf, Bool),
    `duration_quantiles` AggregateFunction(quantiles(0.5, 0.95, 0.99), UInt64)
)
ENGINE = ReplicatedAggregatingMergeTree
PARTITION BY toYYYYMM(bucket_start)
ORDER BY (tenant_id, service_name, operation_name, bucket_start)
TTL bucket_start + toIntervalDay(365)
SETTINGS index_granularity = 8192;

CREATE MATERIALIZED VIEW red_metrics_1h_mv TO red_metrics_1h AS
SELECT
    tenant_id, service_name, operation_name,
    toStartOfHour(bucket_start) AS bucket_start,
    countMergeState(request_count) AS request_count,
    countIfMergeState(error_count) AS error_count,
    quantilesMergeState(0.5, 0.95, 0.99)(duration_quantiles) AS duration_quantiles
FROM red_metrics_1m
GROUP BY tenant_id, service_name, operation_name, bucket_start;

-- ============================================================
-- SERVICE DEPENDENCY AGGREGATES (from spans)
-- ============================================================

CREATE TABLE service_dependencies_agg
(
    `tenant_id`         LowCardinality(String),
    `source_service`    LowCardinality(String),
    `target_service`    LowCardinality(String),
    `protocol`          LowCardinality(String),
    `bucket_start`      DateTime CODEC(DoubleDelta, LZ4),
    `call_count`        AggregateFunction(count, UInt64),
    `error_count`       AggregateFunction(countIf, Bool),
    `duration_quantiles` AggregateFunction(quantiles(0.5, 0.99), UInt64)
)
ENGINE = ReplicatedAggregatingMergeTree
PARTITION BY toYYYYMM(bucket_start)
ORDER BY (tenant_id, source_service, target_service, bucket_start)
SETTINGS index_granularity = 8192;

CREATE MATERIALIZED VIEW service_deps_mv TO service_dependencies_agg AS
SELECT
    tenant_id,
    resources_string['service.name'] AS source_service,
    attributes_string['peer.service'] AS target_service,
    multiIf(
        rpc_system != '', rpc_system,
        db_system != '', db_system,
        messaging_system != '', messaging_system,
        http_method != '', 'http',
        'unknown'
    ) AS protocol,
    toStartOfFiveMinutes(timestamp) AS bucket_start,
    countState() AS call_count,
    countIfState(has_error) AS error_count,
    quantilesState(0.5, 0.99)(duration_nano) AS duration_quantiles
FROM spans
WHERE kind = 3  -- CLIENT spans represent outgoing calls
  AND attributes_string['peer.service'] != ''
GROUP BY tenant_id, source_service, target_service, protocol, bucket_start;
```

## ClickHouse: Error Index

```sql
-- ============================================================
-- CLICKHOUSE: ERROR INDEX
-- ============================================================
-- Dedicated table for exception details derived from span events.

CREATE TABLE error_index
(
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `error_id`              String CODEC(ZSTD(1)),
    `group_id`              String CODEC(ZSTD(1)),  -- hash of exception type + message
    `trace_id`              FixedString(32) CODEC(ZSTD(1)),
    `span_id`               String CODEC(ZSTD(1)),
    `service_name`          LowCardinality(String) CODEC(ZSTD(1)),
    `exception_type`        LowCardinality(String) CODEC(ZSTD(1)),
    `exception_message`     String CODEC(ZSTD(2)),
    `exception_stacktrace`  String CODEC(ZSTD(3)),
    `exception_escaped`     Bool CODEC(ZSTD(1)),
    `resource_attributes`   Map(LowCardinality(String), String) CODEC(ZSTD(1))
)
ENGINE = ReplicatedMergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, service_name, group_id, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(30)
SETTINGS index_granularity = 8192;
```

## ClickHouse: Attribute Catalog

```sql
-- ============================================================
-- CLICKHOUSE: ATTRIBUTE CATALOG (for UI autocomplete)
-- ============================================================

CREATE TABLE span_attribute_keys
(
    `tenant_id`     LowCardinality(String),
    `tag_key`       LowCardinality(String),
    `tag_type`      Enum8('tag' = 1, 'resource' = 2),
    `data_type`     Enum8('string' = 1, 'bool' = 2, 'float64' = 3),
    `is_column`     Bool  -- true if materialized as a top-level column
)
ENGINE = ReplicatedReplacingMergeTree
ORDER BY (tenant_id, tag_key, tag_type, data_type)
SETTINGS index_granularity = 8192;

CREATE TABLE top_level_operations
(
    `tenant_id`     LowCardinality(String),
    `service_name`  LowCardinality(String),
    `name`          LowCardinality(String),
    `kind`          LowCardinality(String)
)
ENGINE = ReplicatedReplacingMergeTree
ORDER BY (tenant_id, service_name, name, kind)
SETTINGS index_granularity = 8192;
```

## Example Queries

```sql
-- ============================================================
-- EXAMPLE: Trace reconstruction (ClickHouse)
-- ============================================================
SELECT
    trace_id, span_id, parent_span_id, name, kind_string,
    duration_nano / 1000000 AS duration_ms,
    status_code, status_message,
    attributes_string, resources_string['service.name'] AS service
FROM spans
WHERE tenant_id = 'tenant-123'
  AND trace_id = '0af7651916cd43dd8448eb211c80319c'
ORDER BY timestamp;

-- ============================================================
-- EXAMPLE: RED metrics dashboard query (ClickHouse)
-- ============================================================
SELECT
    bucket_start,
    countMerge(request_count) AS requests,
    countIfMerge(error_count) AS errors,
    round(countIfMerge(error_count) / countMerge(request_count) * 100, 2) AS error_rate_pct,
    quantilesMerge(0.5, 0.95, 0.99)(duration_quantiles) AS latency_percentiles
FROM red_metrics_1m
WHERE tenant_id = 'tenant-123'
  AND service_name = 'order-service'
  AND bucket_start >= now() - INTERVAL 1 HOUR
GROUP BY bucket_start
ORDER BY bucket_start;

-- ============================================================
-- EXAMPLE: Service topology with error rates (ClickHouse)
-- ============================================================
SELECT
    source_service,
    target_service,
    protocol,
    countMerge(call_count) AS total_calls,
    countIfMerge(error_count) AS errors,
    quantilesMerge(0.99)(duration_quantiles) AS p99_latency
FROM service_dependencies_agg
WHERE tenant_id = 'tenant-123'
  AND bucket_start >= now() - INTERVAL 1 HOUR
GROUP BY source_service, target_service, protocol
ORDER BY total_calls DESC;

-- ============================================================
-- EXAMPLE: Resource-filtered trace search (ClickHouse + CTE)
-- Follows SigNoz pattern of fingerprint-based resource filtering
-- ============================================================
WITH matching_resources AS (
    SELECT fingerprint
    FROM traces_resource
    WHERE labels LIKE '%"service.name":"order-service"%'
      AND seen_at_ts_bucket_start >= toUnixTimestamp(now() - INTERVAL 1 HOUR)
)
SELECT trace_id, name, duration_nano / 1000000 AS duration_ms, has_error
FROM spans
WHERE tenant_id = 'tenant-123'
  AND resource_fingerprint IN (SELECT fingerprint FROM matching_resources)
  AND ts_bucket_start >= toUnixTimestamp(now() - INTERVAL 1 HOUR) - 1800
  AND ts_bucket_start <= toUnixTimestamp(now()) + 1800
  AND has_error = true
ORDER BY duration_nano DESC
LIMIT 100;
```

---

## Table Count Summary

| Category | Database | Tables | Notes |
|----------|----------|--------|-------|
| Tenant & Identity | PostgreSQL | 3 | tenants, users, api_keys |
| Service Registry | PostgreSQL | 1 | services |
| SLOs & Alerts | PostgreSQL | 5 | slos, alert_rules, notification_channels, alert_rule_channels, incidents + incident_events |
| AI & Sampling | PostgreSQL | 2 | ai_analyses, sampling_rules |
| Trace Spans | ClickHouse | 2 | spans, traces_resource |
| Metrics | ClickHouse | 1 | metric_samples |
| Logs | ClickHouse | 1 | log_records |
| Network Flows | ClickHouse | 1 | network_flows |
| RED Aggregates | ClickHouse | 2 + 2 MVs | red_metrics_1m, red_metrics_1h + materialized views |
| Service Dependencies | ClickHouse | 1 + 1 MV | service_dependencies_agg + materialized view |
| Error Index | ClickHouse | 1 | error_index |
| Catalog | ClickHouse | 2 | span_attribute_keys, top_level_operations |
| **Total** | **Both** | **22 tables + 3 MVs** | 11 PostgreSQL + 11 ClickHouse |

---

## Key Design Decisions

1. **Dual-database architecture** — PostgreSQL for OLTP (configuration, incidents, SLOs) and ClickHouse for OLAP (telemetry). This is the architecture proven at scale by SigNoz. Each database is independently scalable — add PostgreSQL read replicas for configuration reads, add ClickHouse shards for telemetry volume.

2. **ClickHouse Map columns for OTLP attributes** — Variable-cardinality span attributes are stored as `Map(LowCardinality(String), String)` rather than normalized key-value rows. This avoids the massive write amplification of the relational approach (1 row per attribute per span) while keeping attributes queryable via ClickHouse's map access syntax (`attributes_string['http.method']`).

3. **Materialized top-level columns for hot attributes** — Frequently queried attributes (http_method, http_url, db_system, etc.) are promoted to dedicated columns with LowCardinality encoding. This matches SigNoz's "selected attributes" pattern and provides 5-10x faster filtering than map access for high-cardinality fields.

4. **30-minute timestamp bucketing** — The `ts_bucket_start` column groups spans into 30-minute buckets, matching SigNoz's approach. Queries must pad their time range by +/- 1800 seconds, but in return ClickHouse can skip entire granules when scanning, reducing query latency dramatically.

5. **Resource fingerprint deduplication** — Resource attributes (service.name, host, SDK version) are hashed into a fingerprint and stored once in `traces_resource`. Spans reference this fingerprint. This reduces storage by 60-80% for resource metadata compared to repeating the full resource on every span.

6. **AggregatingMergeTree for RED metrics** — The `red_metrics_1m` table uses `AggregateFunction` columns with the AggregatingMergeTree engine. Materialized views incrementally maintain partial aggregate states (countState, quantilesState) at insert time. Dashboard queries call *Merge functions to finalize aggregates — O(minutes in time range) instead of O(spans in time range).

7. **Per-day partitioning with TTL** — ClickHouse tables are partitioned by day and have configurable TTL (30 days for spans, 90 days for metrics, 365 days for hourly aggregates). Old partitions are automatically dropped, providing zero-cost data retention management.

8. **Bloom filter indexes for trace_id lookup** — ClickHouse's `bloom_filter` skip index on `trace_id` enables fast point lookups by trace ID without maintaining a full secondary index, keeping write performance high while supporting the "find trace by ID" query pattern that accounts for ~40% of observability queries.

9. **Token-based full-text search for logs** — The `tokenbf_v1` skip index on `log_records.body` enables substring search across log messages without a dedicated full-text search engine, keeping the architecture simpler while supporting the "search logs by keyword" pattern.

10. **AI sampling metadata in span columns** — The `sampling_decision` and `novelty_score` columns on the spans table record the AI-based sampling system's decision for each span. This enables analysis of sampling effectiveness ("what percentage of error traces did the AI sampler retain?") and feedback loops for model improvement.
