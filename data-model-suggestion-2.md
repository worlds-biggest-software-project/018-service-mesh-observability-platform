# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Service Mesh Observability Platform · Created: 2026-05-11

## Philosophy

This approach treats the observability platform itself as an event-sourced system: every state change — a span arriving, a metric data point being recorded, an alert firing, an SLO being created, an AI analysis being generated — is captured as an immutable event in a single append-only event store. Materialized read models (projections) are built from these events to serve different query patterns: a trace reconstruction view, a service topology view, a metric dashboard view, an incident timeline view.

This mirrors how systems like Kafka-based observability pipelines (used internally at LinkedIn, Uber, and Netflix) actually work: telemetry data flows through an immutable log (Kafka topics) and is projected into various read-optimized stores. The difference here is that the event log is also the durable store, not just a transport layer. Every event is retained and replayable, meaning the system can reconstruct any past state perfectly — "what did the service topology look like at 14:23 UTC last Tuesday?" — by replaying events up to that timestamp.

The core design principle is **immutability as the source of truth**. No UPDATE or DELETE operations touch the event store. All mutable state lives in projections that can be rebuilt from scratch. This provides a complete, tamper-evident audit trail that satisfies the most stringent compliance requirements (FedRAMP, SOC 2 Type II, HIPAA) and enables temporal queries that relational models struggle with. The trade-off is increased storage (events are never deleted, only archived), eventual consistency between the event store and projections, and higher complexity in the projection/replay infrastructure.

**Best for:** Regulated enterprises requiring tamper-evident audit trails, platforms where temporal queries are critical ("what was the error budget at 3pm?"), and teams already operating Kafka-based data pipelines who want to extend the event-streaming paradigm to observability metadata and configuration.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is recorded with timestamp, actor, and context
- (+) Temporal queries are natural — replay to any point in time to reconstruct past state
- (+) Projections can be rebuilt if corrupted or if a new query pattern is needed
- (+) Natural fit for Kafka-based microservice architectures — the observability platform speaks the same language
- (+) Append-only writes are extremely fast — no row locks, no UPDATE contention
- (-) Eventually consistent — projections lag behind the event store by milliseconds to seconds
- (-) Storage-intensive — events accumulate forever (though cold-tier archival mitigates this)
- (-) Projection infrastructure adds operational complexity (stream processors, state stores)
- (-) Debugging requires understanding both the event store AND the projection logic
- (-) Rebuilding projections from billions of events can take hours

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry Trace Data Model | Span arrival events contain the full OTLP Span proto as a JSONB payload, preserving all fields without schema mapping |
| OpenTelemetry Metrics Data Model | Metric data point events carry the full OTLP MetricDataPoint proto, including histogram buckets and exemplars |
| OpenTelemetry Logs Data Model | Log record events contain the full OTLP LogRecord proto with severity, body, and trace correlation |
| W3C Trace Context Level 1/2 | trace_id and span_id are extracted from events and used as correlation keys across projections |
| CloudEvents (CNCF) | Event envelope format follows CloudEvents v1.0 specification: id, source, type, time, data |
| OCSF (Open Cybersecurity Schema Framework) | Security and audit events follow OCSF categories for incident detection and compliance reporting |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- CORE EVENT STORE — append-only, immutable
-- ============================================================

-- The single source of truth. Every state change in the platform
-- is recorded here as an immutable event. No UPDATE or DELETE
-- operations are permitted on this table.

CREATE TABLE events (
    -- Event identity (CloudEvents-aligned)
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    event_type      TEXT NOT NULL,          -- hierarchical type (see enum below)
    event_source    TEXT NOT NULL,          -- originating component
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Tenant isolation
    tenant_id       UUID NOT NULL,

    -- Aggregate identification (the entity this event belongs to)
    aggregate_type  TEXT NOT NULL,          -- 'trace', 'service', 'slo', 'alert_rule', 'incident'
    aggregate_id    TEXT NOT NULL,          -- the entity's primary identifier

    -- Causation chain
    correlation_id  TEXT,                   -- groups related events (e.g., trace_id)
    causation_id    UUID,                   -- the event that caused this one
    actor_id        UUID,                   -- user or system that produced the event
    actor_type      TEXT DEFAULT 'system'
                    CHECK (actor_type IN ('user', 'system', 'api_key', 'ai')),

    -- Payload — the actual event data as JSONB
    data            JSONB NOT NULL,
    -- Examples by event_type:
    --
    -- "span.received":
    -- {
    --   "trace_id": "abc123...",
    --   "span_id": "def456...",
    --   "parent_span_id": "ghi789...",
    --   "name": "GET /api/orders",
    --   "kind": 2,
    --   "start_time_unix_nano": 1715000000000000000,
    --   "end_time_unix_nano": 1715000000340000000,
    --   "status": {"code": 0},
    --   "attributes": {"http.method": "GET", "http.status_code": 200},
    --   "resource": {"service.name": "order-service", "k8s.namespace": "prod"}
    -- }
    --
    -- "slo.created":
    -- {
    --   "name": "Order API Availability",
    --   "service_name": "order-service",
    --   "target_percent": 99.95,
    --   "window_days": 30,
    --   "sli_type": "availability"
    -- }
    --
    -- "incident.ai_analysis_completed":
    -- {
    --   "incident_id": "...",
    --   "narrative": "Latency on order-service → payment-gateway increased...",
    --   "model": "claude-opus-4-6",
    --   "confidence": 0.87,
    --   "correlated_signals": ["trace:abc123", "metric:p99_latency", "log:deploy_event"]
    -- }

    -- Schema version for forward compatibility
    schema_version  INT NOT NULL DEFAULT 1,

    -- Partitioning key
    PRIMARY KEY (event_id, event_time)
) PARTITION BY RANGE (event_time);

-- Monthly partitions
CREATE TABLE events_2026_05 PARTITION OF events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_2026_06 PARTITION OF events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Immutability enforcement: revoke UPDATE and DELETE
REVOKE UPDATE, DELETE ON events FROM PUBLIC;

-- ============================================================
-- EVENT TYPE TAXONOMY
-- ============================================================
-- Hierarchical event types (dot-separated namespace):
--
-- Telemetry ingestion:
--   span.received
--   metric.received
--   log.received
--   network_flow.received
--
-- Service registry:
--   service.discovered
--   service.updated
--   service.dependency_detected
--   service.operation_detected
--
-- SLO lifecycle:
--   slo.created
--   slo.updated
--   slo.deleted
--   slo.snapshot_computed
--   slo.budget_warning         (error budget < 25%)
--   slo.budget_exhausted       (error budget = 0%)
--   slo.breach_predicted       (AI predictive)
--
-- Alert lifecycle:
--   alert_rule.created
--   alert_rule.updated
--   alert_rule.deleted
--   alert.fired
--   alert.resolved
--
-- Incident lifecycle:
--   incident.created
--   incident.acknowledged
--   incident.escalated
--   incident.comment_added
--   incident.ai_analysis_requested
--   incident.ai_analysis_completed
--   incident.resolved
--   incident.reopened
--
-- Sampling:
--   sampling.rule_created
--   sampling.rule_updated
--   sampling.decision_made
--
-- Configuration:
--   tenant.created
--   tenant.updated
--   user.invited
--   user.role_changed
--   api_key.created
--   api_key.revoked
--   notification_channel.created
--   notification_channel.updated

-- ============================================================
-- INDEXES on event store
-- ============================================================

CREATE INDEX idx_events_tenant_type_time
    ON events(tenant_id, event_type, event_time);

CREATE INDEX idx_events_aggregate
    ON events(tenant_id, aggregate_type, aggregate_id, event_time);

CREATE INDEX idx_events_correlation
    ON events(correlation_id, event_time)
    WHERE correlation_id IS NOT NULL;

-- JSONB path indexes for common telemetry lookups
CREATE INDEX idx_events_trace_id
    ON events((data->>'trace_id'), event_time)
    WHERE event_type = 'span.received';

CREATE INDEX idx_events_service_name
    ON events((data->'resource'->>'service.name'), event_time)
    WHERE event_type IN ('span.received', 'metric.received', 'log.received');
```

## Projection: Trace Reconstruction View

```sql
-- ============================================================
-- PROJECTION: TRACE VIEW (materialized from span.received events)
-- ============================================================
-- This projection is rebuilt by a stream processor consuming
-- span.received events and upserting into these tables.

CREATE TABLE proj_traces (
    trace_id        CHAR(32) NOT NULL,
    tenant_id       UUID NOT NULL,
    root_service    TEXT,
    root_operation  TEXT,
    span_count      INT NOT NULL DEFAULT 0,
    duration_ns     BIGINT,
    has_error       BOOLEAN NOT NULL DEFAULT false,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,          -- watermark: last event processed
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (trace_id, started_at)
) PARTITION BY RANGE (started_at);

CREATE TABLE proj_spans (
    trace_id        CHAR(32) NOT NULL,
    span_id         CHAR(16) NOT NULL,
    parent_span_id  CHAR(16),
    tenant_id       UUID NOT NULL,
    service_name    TEXT NOT NULL,
    operation_name  TEXT NOT NULL,
    kind            SMALLINT NOT NULL,
    start_time_ns   BIGINT NOT NULL,
    end_time_ns     BIGINT NOT NULL,
    duration_ns     BIGINT NOT NULL,
    status_code     SMALLINT NOT NULL DEFAULT 0,
    status_message  TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    resource        JSONB NOT NULL DEFAULT '{}',
    events          JSONB NOT NULL DEFAULT '[]',   -- span events array
    links           JSONB NOT NULL DEFAULT '[]',   -- span links array
    started_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (span_id, trace_id, started_at)
) PARTITION BY RANGE (started_at);

CREATE INDEX idx_proj_spans_trace ON proj_spans(trace_id, started_at);
CREATE INDEX idx_proj_spans_service ON proj_spans(service_name, started_at);
CREATE INDEX idx_proj_spans_duration ON proj_spans(tenant_id, duration_ns, started_at);
CREATE INDEX idx_proj_spans_error ON proj_spans(tenant_id, started_at)
    WHERE status_code = 2;
```

## Projection: Service Topology

```sql
-- ============================================================
-- PROJECTION: SERVICE TOPOLOGY (materialized from service.* events)
-- ============================================================

CREATE TABLE proj_services (
    tenant_id       UUID NOT NULL,
    service_name    TEXT NOT NULL,
    namespace       TEXT,
    cluster         TEXT,
    environment     TEXT NOT NULL DEFAULT 'production',
    language        TEXT,
    otel_sdk_version TEXT,
    resource_attributes JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, service_name, environment)
);

CREATE TABLE proj_service_dependencies (
    tenant_id       UUID NOT NULL,
    source_service  TEXT NOT NULL,
    target_service  TEXT NOT NULL,
    protocol        TEXT,
    call_count_1h   BIGINT NOT NULL DEFAULT 0,
    error_count_1h  BIGINT NOT NULL DEFAULT 0,
    p50_latency_ns  BIGINT,
    p99_latency_ns  BIGINT,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, source_service, target_service, protocol)
);

CREATE TABLE proj_service_operations (
    tenant_id       UUID NOT NULL,
    service_name    TEXT NOT NULL,
    operation_name  TEXT NOT NULL,
    span_kind       TEXT NOT NULL,
    call_count_1h   BIGINT NOT NULL DEFAULT 0,
    error_rate_1h   NUMERIC(5,4),
    p50_latency_ns  BIGINT,
    p95_latency_ns  BIGINT,
    p99_latency_ns  BIGINT,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, service_name, operation_name, span_kind)
);
```

## Projection: SLO & Error Budget State

```sql
-- ============================================================
-- PROJECTION: SLO STATE (materialized from slo.* events)
-- ============================================================

CREATE TABLE proj_slos (
    id              UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    service_name    TEXT NOT NULL,
    name            TEXT NOT NULL,
    sli_type        TEXT NOT NULL,
    target_percent  NUMERIC(6,3) NOT NULL,
    window_type     TEXT NOT NULL,
    window_days     INT NOT NULL,
    sli_query       JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    version         INT NOT NULL DEFAULT 1,    -- incremented on each slo.updated event
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (id)
);

-- Historical SLO state: every version is preserved
CREATE TABLE proj_slo_versions (
    slo_id          UUID NOT NULL,
    version         INT NOT NULL,
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    target_percent  NUMERIC(6,3) NOT NULL,
    sli_query       JSONB NOT NULL,
    changed_by      UUID,
    changed_at      TIMESTAMPTZ NOT NULL,
    event_id        UUID NOT NULL,          -- the slo.updated event
    PRIMARY KEY (slo_id, version)
);

CREATE TABLE proj_slo_snapshots (
    slo_id          UUID NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL,
    window_start    TIMESTAMPTZ NOT NULL,
    window_end      TIMESTAMPTZ NOT NULL,
    good_events     BIGINT NOT NULL,
    total_events    BIGINT NOT NULL,
    sli_value       NUMERIC(8,5) NOT NULL,
    error_budget_remaining NUMERIC(8,5) NOT NULL,
    burn_rate_1h    NUMERIC(8,5),
    burn_rate_6h    NUMERIC(8,5),
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (slo_id, computed_at)
);

-- Temporal query example:
-- "What was the error budget for SLO X at 14:00 UTC on 2026-05-10?"
--
-- SELECT * FROM proj_slo_snapshots
-- WHERE slo_id = 'xxx'
--   AND computed_at <= '2026-05-10T14:00:00Z'
-- ORDER BY computed_at DESC
-- LIMIT 1;
```

## Projection: Incidents & AI Analysis

```sql
-- ============================================================
-- PROJECTION: INCIDENT STATE (materialized from incident.* events)
-- ============================================================

CREATE TABLE proj_incidents (
    id              UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    title           TEXT NOT NULL,
    severity        TEXT NOT NULL,
    status          TEXT NOT NULL,
    triggered_by_rule TEXT,
    ai_summary      TEXT,
    ai_model        TEXT,
    ai_confidence   NUMERIC(4,3),
    affected_services TEXT[] NOT NULL DEFAULT '{}',
    started_at      TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    version         INT NOT NULL DEFAULT 1,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (id)
);

-- Full incident timeline: reconstructed from all incident.* events
-- for a given aggregate_id (incident_id)
CREATE TABLE proj_incident_timeline (
    incident_id     UUID NOT NULL,
    seq             INT NOT NULL,          -- event sequence within incident
    event_type      TEXT NOT NULL,
    actor_id        UUID,
    actor_type      TEXT,
    message         TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL,
    event_id        UUID NOT NULL,          -- reference back to event store
    PRIMARY KEY (incident_id, seq)
);

CREATE INDEX idx_proj_incidents_tenant ON proj_incidents(tenant_id, status, started_at);
```

## Projection: Alert Rules & Notification Channels

```sql
-- ============================================================
-- PROJECTION: CONFIGURATION STATE
-- ============================================================

CREATE TABLE proj_alert_rules (
    id              UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL,
    service_name    TEXT,
    slo_id          UUID,
    condition       JSONB NOT NULL,
    severity        TEXT NOT NULL,
    channel_ids     UUID[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    version         INT NOT NULL DEFAULT 1,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE proj_notification_channels (
    id              UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    channel_type    TEXT NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    version         INT NOT NULL DEFAULT 1,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE proj_tenants (
    id              UUID NOT NULL,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL,
    retention_days  INT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE proj_users (
    id              UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    email           TEXT NOT NULL,
    display_name    TEXT,
    role            TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (id)
);
```

## Projection: RED Metrics (Pre-Aggregated)

```sql
-- ============================================================
-- PROJECTION: RED METRICS (Rate, Error, Duration per service per minute)
-- ============================================================
-- Computed by a stream processor consuming span.received events
-- and aggregating into 1-minute buckets.

CREATE TABLE proj_red_metrics (
    tenant_id       UUID NOT NULL,
    service_name    TEXT NOT NULL,
    operation_name  TEXT NOT NULL,
    bucket_start    TIMESTAMPTZ NOT NULL,  -- 1-minute bucket
    request_count   BIGINT NOT NULL DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    duration_sum_ns BIGINT NOT NULL DEFAULT 0,
    duration_min_ns BIGINT,
    duration_max_ns BIGINT,
    -- Approximate percentile sketches stored as JSONB
    duration_histogram JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"buckets": [1000000, 5000000, 10000000, 50000000, 100000000, 500000000],
    --  "counts":  [1234,    5678,    2345,     890,      123,       12]}
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, service_name, operation_name, bucket_start)
) PARTITION BY RANGE (bucket_start);

-- Hourly rollup
CREATE TABLE proj_red_metrics_hourly (
    tenant_id       UUID NOT NULL,
    service_name    TEXT NOT NULL,
    operation_name  TEXT NOT NULL,
    bucket_start    TIMESTAMPTZ NOT NULL,  -- 1-hour bucket
    request_count   BIGINT NOT NULL DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    duration_sum_ns BIGINT NOT NULL DEFAULT 0,
    duration_min_ns BIGINT,
    duration_max_ns BIGINT,
    duration_p50_ns BIGINT,
    duration_p95_ns BIGINT,
    duration_p99_ns BIGINT,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (tenant_id, service_name, operation_name, bucket_start)
) PARTITION BY RANGE (bucket_start);

CREATE INDEX idx_red_metrics_service ON proj_red_metrics(service_name, bucket_start);
CREATE INDEX idx_red_hourly_service ON proj_red_metrics_hourly(service_name, bucket_start);
```

## Stream Processor Checkpoints

```sql
-- ============================================================
-- STREAM PROCESSOR STATE
-- ============================================================
-- Each projection processor tracks its position in the event stream.

CREATE TABLE projection_checkpoints (
    projection_name TEXT NOT NULL,         -- e.g. "trace_view", "topology", "red_metrics"
    tenant_id       UUID NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    lag_ms          INT,                   -- current lag behind event store
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, tenant_id)
);

-- Dead letter queue for events that fail projection processing
CREATE TABLE projection_dead_letters (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    projection_name TEXT NOT NULL,
    event_id        UUID NOT NULL,
    event_type      TEXT NOT NULL,
    tenant_id       UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INT NOT NULL DEFAULT 0,
    max_retries     INT NOT NULL DEFAULT 3,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_retried_at TIMESTAMPTZ
);

CREATE INDEX idx_dead_letters_projection ON projection_dead_letters(projection_name, created_at);
```

## Example: Event Replay Query

```sql
-- Reconstruct the state of an SLO at a specific point in time
-- by replaying all events for that aggregate up to the target timestamp.

WITH slo_events AS (
    SELECT
        event_type,
        data,
        event_time,
        ROW_NUMBER() OVER (ORDER BY event_time) AS seq
    FROM events
    WHERE tenant_id = 'tenant-uuid'
      AND aggregate_type = 'slo'
      AND aggregate_id = 'slo-uuid'
      AND event_time <= '2026-05-10T14:00:00Z'
    ORDER BY event_time
)
SELECT
    -- The latest slo.created or slo.updated event gives us the config
    (SELECT data FROM slo_events
     WHERE event_type IN ('slo.created', 'slo.updated')
     ORDER BY seq DESC LIMIT 1) AS current_config,

    -- The latest slo.snapshot_computed gives us the error budget
    (SELECT data FROM slo_events
     WHERE event_type = 'slo.snapshot_computed'
     ORDER BY seq DESC LIMIT 1) AS latest_snapshot,

    -- Count of all events for this SLO
    (SELECT COUNT(*) FROM slo_events) AS total_events;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only events table (partitioned) |
| Projection: Traces | 2 | proj_traces, proj_spans |
| Projection: Topology | 3 | proj_services, proj_service_dependencies, proj_service_operations |
| Projection: SLOs | 3 | proj_slos, proj_slo_versions, proj_slo_snapshots |
| Projection: Incidents | 2 | proj_incidents, proj_incident_timeline |
| Projection: Config | 4 | proj_alert_rules, proj_notification_channels, proj_tenants, proj_users |
| Projection: RED Metrics | 2 | proj_red_metrics, proj_red_metrics_hourly |
| Stream Infrastructure | 2 | projection_checkpoints, projection_dead_letters |
| **Total** | **19** | Plus partition tables (auto-generated) |

---

## Key Design Decisions

1. **Single event store table** — All events flow into one `events` table, regardless of type. This simplifies the write path (one INSERT per event), enables cross-aggregate correlation queries, and ensures a single, total ordering of events per tenant. The `event_type` taxonomy provides structure within the flat table.

2. **CloudEvents envelope format** — Event metadata fields (`event_id`, `event_type`, `event_source`, `event_time`) follow the CloudEvents v1.0 specification, making the event store interoperable with Kafka, CloudEvents-compatible message brokers, and CNCF ecosystem tooling.

3. **JSONB payload with schema versioning** — Event data is stored as JSONB with a `schema_version` integer. This allows the event schema to evolve without ALTER TABLE migrations. Old events retain their original schema; projection processors handle version-specific deserialization. OTLP span, metric, and log protos are stored verbatim in the JSONB payload.

4. **Projection tables are disposable** — Every `proj_*` table can be dropped and rebuilt from the event store. The `last_event_id` watermark in each projection row tracks the latest event that contributed to that row, enabling incremental rebuilds and consistency verification.

5. **Aggregate-based partitioning of events** — The `aggregate_type` + `aggregate_id` fields enable efficient replay of a single entity's history (e.g., all events for SLO "X"). Combined with the `correlation_id` (set to `trace_id` for telemetry events), this supports both entity-centric and trace-centric event retrieval.

6. **Immutability enforcement at the database level** — `REVOKE UPDATE, DELETE ON events FROM PUBLIC` prevents any code path from mutating historical events. This is a hard guarantee for compliance audits, not just a convention.

7. **Pre-aggregated RED metrics as a projection** — Rather than querying raw span events for dashboards, a stream processor maintains 1-minute and 1-hour RED metric rollups. This transforms O(millions) span reads into O(thousands) pre-aggregated reads for dashboard queries.

8. **Dead letter queue for projection failures** — Events that fail to project (malformed data, schema version mismatch) are captured in `projection_dead_letters` rather than blocking the stream processor. This ensures the system degrades gracefully — dashboards may show slightly stale data rather than stopping entirely.

9. **Temporal queries via event replay** — The ability to answer "what was true at time T?" is a natural consequence of the immutable event store. The example SQL shows how to reconstruct SLO state at any historical timestamp by replaying events, which is impossible in a mutable relational model without explicit snapshotting.

10. **Version tracking on projections** — Configuration entities (SLOs, alert rules, tenants) include a `version` counter incremented on each update event. Combined with `proj_slo_versions`, this provides a complete change history without querying the event store directly.
