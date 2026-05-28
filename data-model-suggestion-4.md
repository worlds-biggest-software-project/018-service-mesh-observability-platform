# Data Model Suggestion 4: Graph-Relational (PostgreSQL + Property Graph)

> Project: Service Mesh Observability Platform · Created: 2026-05-11

## Philosophy

This approach places the service topology graph at the center of the data model. In a service mesh observability platform, the most valuable insights come from understanding relationships: which service calls which, how errors propagate through dependency chains, what the blast radius of a failing service is, and how the topology has changed over time. A graph-first model makes these relationship-heavy queries natural and performant, while relational tables handle the operational CRUD that graphs are poor at (user management, alert rule configuration, SLO definitions).

The architecture uses PostgreSQL as the single database engine but implements a property graph layer using dedicated `graph_nodes` and `graph_edges` tables with JSONB properties. This is sometimes called the "graph-in-SQL" pattern (used by Apache AGE, a PostgreSQL extension, and similar to the approach Neo4j's network management examples demonstrate). The graph layer stores service topology, dependency relationships, and incident propagation paths. Time-partitioned telemetry tables store spans, metrics, and logs with standard columnar techniques. The graph and telemetry layers are linked by shared identifiers (service names, trace IDs).

The core design principle is **relationships are first-class citizens**. Instead of discovering topology by joining span tables and grouping by service name (expensive at scale), the graph maintains a live, queryable representation of the service mesh topology with typed edges (CALLS, DEPENDS_ON, CAUSED_BY, ROUTES_TO) and rich properties on both nodes and edges. This enables queries like "find all services within 3 hops of the failing payment-gateway" in milliseconds via recursive CTEs, which would require scanning millions of spans in a flat relational model.

**Best for:** Platforms where topology reasoning is the primary value proposition — blast radius analysis, dependency-aware root cause analysis, change impact assessment, and the AI-native features (topology inference, mesh governance auditing) that require graph traversal. Also strong for organizations migrating from or integrating with Neo4j or similar graph databases.

**Trade-offs:**
- (+) Topology queries (shortest path, blast radius, N-hop neighbors) are natural and fast
- (+) Temporal graph versioning enables "what did the topology look like last week?"
- (+) Incident propagation paths are modeled explicitly as graph edges
- (+) Single database engine (PostgreSQL) simplifies operations compared to dual-database
- (+) Recursive CTEs handle arbitrary-depth traversals without application-level logic
- (-) Graph-in-SQL is less performant than native graph databases (Neo4j) for deep traversals (>5 hops)
- (-) The graph_nodes/graph_edges pattern is unfamiliar to most backend engineers
- (-) JSONB properties on graph elements lack schema enforcement (no CHECK constraints on property keys)
- (-) PostgreSQL still hits throughput ceilings for high-volume telemetry ingestion
- (-) More complex write path — every span arrival potentially updates graph edges

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry Trace Data Model | Span table fields map to OTLP Span proto; service names extracted from resource attributes populate graph nodes |
| OpenTelemetry Metrics Data Model | Metric data points stored in time-partitioned tables with OTLP field mapping |
| OpenTelemetry Logs Data Model | Log records follow OTLP LogRecord schema with trace/span correlation |
| W3C Trace Context Level 1/2 | trace_id and span_id stored as hex strings per the W3C specification |
| Prometheus / OpenMetrics | Metric names and label conventions follow OpenMetrics naming rules |
| ISO/IEC 11179 (Metadata Registry) | Graph node and edge type taxonomies follow controlled vocabulary patterns |
| Apache TinkerPop Gremlin Concepts | Graph model (nodes with labels, edges with labels, properties on both) is conceptually compatible with TinkerPop's property graph model |

---

## Tenant & Identity

```sql
-- ============================================================
-- TENANT & IDENTITY (standard relational)
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
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

-- Row-Level Security
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON api_keys
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

## Property Graph Layer

```sql
-- ============================================================
-- PROPERTY GRAPH LAYER
-- ============================================================
-- A general-purpose property graph implemented in PostgreSQL.
-- Nodes and edges have typed labels and JSONB property bags.
-- This is the heart of the topology model.

-- -------------------------------------------------------
-- GRAPH NODES
-- -------------------------------------------------------
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    label           TEXT NOT NULL,
    -- Node labels (controlled vocabulary):
    --   'service'       - a microservice in the mesh
    --   'operation'     - a specific endpoint/operation on a service
    --   'infrastructure'- a database, cache, queue, or external API
    --   'cluster'       - a Kubernetes cluster
    --   'namespace'     - a Kubernetes namespace
    --   'deployment'    - a specific deployment/release
    --   'incident'      - an active or historical incident
    --   'slo'           - a service level objective

    name            TEXT NOT NULL,         -- human-readable identifier
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties for label='service':
    -- {
    --   "namespace": "production",
    --   "cluster": "us-east-1",
    --   "environment": "production",
    --   "language": "go",
    --   "framework": "gin",
    --   "otel_sdk_version": "1.28.0",
    --   "k8s_deployment": "order-service",
    --   "replicas": 3
    -- }
    --
    -- Example properties for label='infrastructure':
    -- {
    --   "type": "postgresql",
    --   "version": "16.2",
    --   "host": "payment-db.internal",
    --   "port": 5432
    -- }
    --
    -- Example properties for label='incident':
    -- {
    --   "severity": "critical",
    --   "status": "investigating",
    --   "ai_summary": "Connection pool exhaustion on payment-db...",
    --   "started_at": "2026-05-11T14:23:00Z"
    -- }

    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (tenant_id, label, name)
);

ALTER TABLE graph_nodes ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON graph_nodes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE INDEX idx_nodes_tenant_label ON graph_nodes(tenant_id, label);
CREATE INDEX idx_nodes_properties ON graph_nodes USING GIN (properties);
CREATE INDEX idx_nodes_name ON graph_nodes(tenant_id, name);

-- -------------------------------------------------------
-- GRAPH EDGES
-- -------------------------------------------------------
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id),
    target_id       UUID NOT NULL REFERENCES graph_nodes(id),
    label           TEXT NOT NULL,
    -- Edge labels (controlled vocabulary):
    --   'CALLS'           - service A calls service B (derived from CLIENT spans)
    --   'HOSTS'           - cluster/namespace hosts service
    --   'EXPOSES'         - service exposes operation
    --   'DEPENDS_ON'      - service depends on infrastructure
    --   'ROUTES_TO'       - Istio/Envoy routing configuration
    --   'CAUSED_BY'       - incident was caused by another incident/event
    --   'AFFECTS'         - incident affects service
    --   'MONITORS'        - SLO monitors service
    --   'DEPLOYED_AS'     - service is deployed as deployment

    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties for label='CALLS':
    -- {
    --   "protocol": "grpc",
    --   "call_count_1h": 45230,
    --   "error_count_1h": 12,
    --   "error_rate_1h": 0.0003,
    --   "p50_latency_ms": 2.3,
    --   "p99_latency_ms": 45.7,
    --   "discovered_via": "trace"
    -- }
    --
    -- Example properties for label='CAUSED_BY':
    -- {
    --   "confidence": 0.87,
    --   "evidence": ["trace:abc123", "metric:conn_pool_exhaustion"],
    --   "narrative": "Connection pool exhaustion propagated..."
    -- }
    --
    -- Example properties for label='ROUTES_TO':
    -- {
    --   "mesh": "istio",
    --   "virtual_service": "order-service-vs",
    --   "weight": 100,
    --   "timeout_ms": 5000,
    --   "retries": 3
    -- }

    weight          DOUBLE PRECISION DEFAULT 1.0,  -- for weighted graph algorithms
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Prevent duplicate edges of the same type
    UNIQUE (tenant_id, source_id, target_id, label)
);

ALTER TABLE graph_edges ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON graph_edges
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE INDEX idx_edges_source ON graph_edges(source_id) WHERE is_active = true;
CREATE INDEX idx_edges_target ON graph_edges(target_id) WHERE is_active = true;
CREATE INDEX idx_edges_label ON graph_edges(tenant_id, label) WHERE is_active = true;
CREATE INDEX idx_edges_properties ON graph_edges USING GIN (properties);

-- -------------------------------------------------------
-- GRAPH SNAPSHOTS (temporal graph versioning)
-- -------------------------------------------------------
-- Captures the state of nodes and edges at regular intervals
-- to enable temporal topology queries ("what did the mesh look like last week?")

CREATE TABLE graph_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    snapshot_time   TIMESTAMPTZ NOT NULL,
    node_count      INT NOT NULL,
    edge_count      INT NOT NULL,
    nodes           JSONB NOT NULL,  -- array of {id, label, name, properties}
    edges           JSONB NOT NULL,  -- array of {id, source_id, target_id, label, properties}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshots_tenant_time ON graph_snapshots(tenant_id, snapshot_time);
```

## Graph Query Examples

```sql
-- ============================================================
-- GRAPH QUERY: Blast radius — find all services within N hops
-- of a failing service
-- ============================================================
WITH RECURSIVE blast_radius AS (
    -- Start from the failing service
    SELECT
        n.id, n.name, n.label, n.properties,
        0 AS depth,
        ARRAY[n.id] AS path
    FROM graph_nodes n
    WHERE n.tenant_id = 'tenant-uuid'
      AND n.label = 'service'
      AND n.name = 'payment-gateway'

    UNION ALL

    -- Traverse outgoing CALLS edges (callers of the failing service)
    SELECT
        n2.id, n2.name, n2.label, n2.properties,
        br.depth + 1,
        br.path || n2.id
    FROM blast_radius br
    JOIN graph_edges e ON e.target_id = br.id
        AND e.label = 'CALLS'
        AND e.is_active = true
    JOIN graph_nodes n2 ON n2.id = e.source_id
    WHERE br.depth < 3                      -- limit to 3 hops
      AND n2.id != ALL(br.path)              -- prevent cycles
)
SELECT name, label, depth, properties
FROM blast_radius
ORDER BY depth, name;

-- ============================================================
-- GRAPH QUERY: Service dependency chain with latency
-- ============================================================
SELECT
    src.name AS source_service,
    tgt.name AS target_service,
    e.label,
    (e.properties->>'protocol')::TEXT AS protocol,
    (e.properties->>'p99_latency_ms')::FLOAT AS p99_latency_ms,
    (e.properties->>'error_rate_1h')::FLOAT AS error_rate
FROM graph_edges e
JOIN graph_nodes src ON src.id = e.source_id
JOIN graph_nodes tgt ON tgt.id = e.target_id
WHERE e.tenant_id = 'tenant-uuid'
  AND e.label = 'CALLS'
  AND e.is_active = true
ORDER BY (e.properties->>'call_count_1h')::INT DESC;

-- ============================================================
-- GRAPH QUERY: Incident propagation — trace causal chain
-- ============================================================
WITH RECURSIVE causal_chain AS (
    SELECT
        n.id, n.name, n.properties,
        0 AS depth,
        ARRAY[n.id] AS path
    FROM graph_nodes n
    WHERE n.id = 'incident-uuid'

    UNION ALL

    SELECT
        n2.id, n2.name, n2.properties,
        cc.depth + 1,
        cc.path || n2.id
    FROM causal_chain cc
    JOIN graph_edges e ON e.source_id = cc.id
        AND e.label = 'CAUSED_BY'
    JOIN graph_nodes n2 ON n2.id = e.target_id
    WHERE cc.depth < 10
      AND n2.id != ALL(cc.path)
)
SELECT name, depth, properties->>'ai_summary' AS summary
FROM causal_chain
ORDER BY depth;

-- ============================================================
-- GRAPH QUERY: Mesh governance — find services with
-- misconfigured retry policies
-- ============================================================
SELECT
    src.name AS service,
    tgt.name AS target,
    (e.properties->>'retries')::INT AS retries,
    (e.properties->>'timeout_ms')::INT AS timeout_ms,
    e.properties->>'mesh' AS mesh
FROM graph_edges e
JOIN graph_nodes src ON src.id = e.source_id
JOIN graph_nodes tgt ON tgt.id = e.target_id
WHERE e.tenant_id = 'tenant-uuid'
  AND e.label = 'ROUTES_TO'
  AND e.is_active = true
  AND (e.properties->>'retries')::INT > 5  -- anti-pattern: excessive retries
ORDER BY retries DESC;
```

## Telemetry: Traces & Spans

```sql
-- ============================================================
-- TRACES & SPANS (time-partitioned relational)
-- ============================================================

CREATE TABLE spans (
    trace_id        CHAR(32) NOT NULL,
    span_id         CHAR(16) NOT NULL,
    parent_span_id  CHAR(16),
    tenant_id       UUID NOT NULL,
    service_node_id UUID REFERENCES graph_nodes(id), -- FK to graph!
    name            TEXT NOT NULL,
    kind            SMALLINT NOT NULL DEFAULT 0,
    start_time_ns   BIGINT NOT NULL,
    end_time_ns     BIGINT NOT NULL,
    duration_ns     BIGINT GENERATED ALWAYS AS (end_time_ns - start_time_ns) STORED,
    status_code     SMALLINT NOT NULL DEFAULT 0,
    status_message  TEXT,
    has_error       BOOLEAN GENERATED ALWAYS AS (status_code = 2) STORED,
    attributes      JSONB NOT NULL DEFAULT '{}',
    resource        JSONB NOT NULL DEFAULT '{}',
    events          JSONB NOT NULL DEFAULT '[]',
    links           JSONB NOT NULL DEFAULT '[]',
    started_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (span_id, trace_id, started_at)
) PARTITION BY RANGE (started_at);

CREATE TABLE spans_2026_05 PARTITION OF spans
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Link spans directly to graph nodes for topology-aware queries
CREATE INDEX idx_spans_service_node ON spans(service_node_id, started_at);
CREATE INDEX idx_spans_trace ON spans(trace_id, started_at);
CREATE INDEX idx_spans_duration ON spans(tenant_id, duration_ns, started_at);
CREATE INDEX idx_spans_error ON spans(tenant_id, started_at) WHERE has_error = true;
CREATE INDEX idx_spans_attrs ON spans USING GIN (attributes);
```

## Telemetry: Metrics & Logs

```sql
-- ============================================================
-- METRICS (time-partitioned)
-- ============================================================

CREATE TABLE metric_descriptors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_node_id UUID REFERENCES graph_nodes(id),  -- FK to graph!
    name            TEXT NOT NULL,
    description     TEXT,
    unit            TEXT,
    type            TEXT NOT NULL
                    CHECK (type IN ('gauge', 'sum', 'histogram', 'exp_histogram', 'summary')),
    aggregation_temporality TEXT,
    is_monotonic    BOOLEAN,
    UNIQUE (tenant_id, name, type)
);

CREATE TABLE metric_data_points (
    metric_id       UUID NOT NULL REFERENCES metric_descriptors(id),
    tenant_id       UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    value_double    DOUBLE PRECISION,
    count           BIGINT,
    sum             DOUBLE PRECISION,
    min             DOUBLE PRECISION,
    max             DOUBLE PRECISION,
    bucket_counts   BIGINT[],
    explicit_bounds DOUBLE PRECISION[],
    labels          JSONB NOT NULL DEFAULT '{}',
    exemplar_trace_id CHAR(32),
    exemplar_span_id  CHAR(16),
    PRIMARY KEY (metric_id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE TABLE metric_data_points_2026_05 PARTITION OF metric_data_points
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- ============================================================
-- LOGS (time-partitioned)
-- ============================================================

CREATE TABLE log_records (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id       UUID NOT NULL,
    service_node_id UUID REFERENCES graph_nodes(id),  -- FK to graph!
    timestamp       TIMESTAMPTZ NOT NULL,
    severity_number SMALLINT NOT NULL DEFAULT 0,
    severity_text   TEXT,
    body            TEXT,
    trace_id        CHAR(32),
    span_id         CHAR(16),
    attributes      JSONB NOT NULL DEFAULT '{}',
    resource        JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE TABLE log_records_2026_05 PARTITION OF log_records
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_logs_service_node ON log_records(service_node_id, timestamp);
CREATE INDEX idx_logs_trace ON log_records(trace_id, timestamp) WHERE trace_id IS NOT NULL;
CREATE INDEX idx_logs_severity ON log_records(tenant_id, severity_number, timestamp);
CREATE INDEX idx_logs_body_search ON log_records USING GIN (to_tsvector('english', body));
```

## SLOs, Alerts & Incidents (Graph-Integrated)

```sql
-- ============================================================
-- SLOs (linked to graph nodes)
-- ============================================================

CREATE TABLE slos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_node_id UUID NOT NULL REFERENCES graph_nodes(id),  -- the service this SLO monitors
    slo_node_id     UUID REFERENCES graph_nodes(id),           -- the SLO's own graph node
    name            TEXT NOT NULL,
    sli_type        TEXT NOT NULL
                    CHECK (sli_type IN ('availability', 'latency', 'throughput', 'error_rate')),
    target_percent  NUMERIC(6,3) NOT NULL,
    window_type     TEXT NOT NULL DEFAULT 'rolling',
    window_days     INT NOT NULL DEFAULT 30,
    sli_query       JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- When an SLO is created, a graph node (label='slo') and a
-- MONITORS edge (slo -> service) are also created:
--
-- INSERT INTO graph_nodes (tenant_id, label, name, properties)
-- VALUES ('tenant-uuid', 'slo', 'Order API Availability',
--         '{"target": 99.95, "sli_type": "availability"}');
--
-- INSERT INTO graph_edges (tenant_id, source_id, target_id, label, properties)
-- VALUES ('tenant-uuid', <slo_node_id>, <service_node_id>, 'MONITORS',
--         '{"target_percent": 99.95}');

CREATE TABLE slo_snapshots (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    slo_id          UUID NOT NULL REFERENCES slos(id),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    window_start    TIMESTAMPTZ NOT NULL,
    window_end      TIMESTAMPTZ NOT NULL,
    good_events     BIGINT NOT NULL,
    total_events    BIGINT NOT NULL,
    sli_value       NUMERIC(8,5) NOT NULL,
    error_budget_remaining NUMERIC(8,5) NOT NULL,
    burn_rate_1h    NUMERIC(8,5),
    burn_rate_6h    NUMERIC(8,5)
);

-- ============================================================
-- ALERT RULES & NOTIFICATION
-- ============================================================

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

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('threshold', 'anomaly', 'slo_burn_rate',
                           'predictive', 'topology_anomaly')),
                    -- 'topology_anomaly' is unique to this model: detects
                    -- new/missing edges or unusual edge weight changes
    service_node_id UUID REFERENCES graph_nodes(id),
    slo_id          UUID REFERENCES slos(id),
    condition       JSONB NOT NULL,
    -- Example condition for topology_anomaly:
    -- {
    --   "type": "new_dependency",
    --   "description": "Alert when a service starts calling a new dependency"
    -- }
    -- {
    --   "type": "edge_weight_change",
    --   "edge_label": "CALLS",
    --   "property": "error_rate_1h",
    --   "threshold": 0.05,
    --   "comparison": ">"
    -- }
    severity        TEXT NOT NULL DEFAULT 'warning',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rule_channels (
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id),
    channel_id      UUID NOT NULL REFERENCES notification_channels(id),
    PRIMARY KEY (alert_rule_id, channel_id)
);

-- ============================================================
-- INCIDENTS (graph-native)
-- ============================================================
-- Incidents are BOTH relational records AND graph nodes.
-- The graph representation enables causal chain traversal.

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    incident_node_id UUID REFERENCES graph_nodes(id),  -- the incident's graph node
    title           TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning',
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'acknowledged', 'investigating', 'resolved')),
    triggered_by_rule_id UUID REFERENCES alert_rules(id),
    ai_summary      TEXT,
    ai_model        TEXT,
    ai_confidence   NUMERIC(4,3),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- When an incident is created:
-- 1. A graph_nodes entry is created (label='incident')
-- 2. AFFECTS edges are created from incident -> each affected service
-- 3. CAUSED_BY edges are created when AI identifies root cause
-- 4. These graph edges enable traversal queries:
--    "Show me all incidents that affected services in the payment namespace"
--    "Trace the causal chain from this customer-facing error to the root cause"

CREATE TABLE incident_events (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    incident_id     UUID NOT NULL REFERENCES incidents(id),
    event_type      TEXT NOT NULL,
    user_id         UUID REFERENCES users(id),
    message         TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_slos_service_node ON slos(service_node_id);
CREATE INDEX idx_slo_snapshots_slo ON slo_snapshots(slo_id, computed_at);
CREATE INDEX idx_incidents_tenant_status ON incidents(tenant_id, status, started_at);
CREATE INDEX idx_incident_events_incident ON incident_events(incident_id);
```

## AI Analysis & Sampling

```sql
-- ============================================================
-- AI ANALYSIS (graph-enriched)
-- ============================================================

CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    incident_id     UUID REFERENCES incidents(id),
    analysis_type   TEXT NOT NULL
                    CHECK (analysis_type IN ('root_cause', 'prediction',
                           'governance_audit', 'topology_inference')),
                    -- 'topology_inference' is unique to this model:
                    -- AI-inferred graph edges from eBPF/traffic patterns
    input_signals   JSONB NOT NULL,
    output_narrative TEXT NOT NULL,

    -- Graph mutations proposed by this analysis
    proposed_graph_changes JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"action": "add_edge", "source": "order-service", "target": "payment-db",
    --    "label": "DEPENDS_ON", "confidence": 0.92},
    --   {"action": "add_edge", "source": "incident-123", "target": "payment-db-overload",
    --    "label": "CAUSED_BY", "confidence": 0.87}
    -- ]
    graph_changes_applied BOOLEAN NOT NULL DEFAULT false,

    model_id        TEXT NOT NULL,
    confidence      NUMERIC(4,3),
    feedback_rating SMALLINT,
    tokens_used     INT,
    latency_ms      INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SAMPLING RULES
-- ============================================================

CREATE TABLE sampling_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_node_id UUID REFERENCES graph_nodes(id),
    rule_type       TEXT NOT NULL
                    CHECK (rule_type IN ('always_sample', 'never_sample',
                           'rate_limit', 'ai_adaptive', 'topology_aware')),
                    -- 'topology_aware' is unique to this model:
                    -- sampling priority based on graph position
                    -- (e.g., always sample spans from services with high fan-out)
    condition       JSONB NOT NULL,
    -- Example for topology_aware:
    -- {
    --   "min_downstream_services": 5,
    --   "sample_rate": 1.0,
    --   "description": "Always sample spans from hub services with 5+ downstream dependencies"
    -- }
    priority        INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_analyses_incident ON ai_analyses(incident_id);
CREATE INDEX idx_sampling_service_node ON sampling_rules(service_node_id);
```

## eBPF Network Flows

```sql
-- ============================================================
-- eBPF NETWORK FLOWS (feeds graph edge creation)
-- ============================================================

CREATE TABLE network_flows (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id       UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    source_node_id  UUID REFERENCES graph_nodes(id),  -- FK to graph!
    target_node_id  UUID REFERENCES graph_nodes(id),  -- FK to graph!
    source_pod      TEXT,
    dest_pod        TEXT,
    protocol        TEXT NOT NULL,
    l7_protocol     TEXT,
    dest_port       INT,
    http_method     TEXT,
    http_status     INT,
    dns_query       TEXT,
    bytes_sent      BIGINT DEFAULT 0,
    bytes_received  BIGINT DEFAULT 0,
    verdict         TEXT NOT NULL DEFAULT 'allowed',
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE TABLE network_flows_2026_05 PARTITION OF network_flows
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Network flows feed the graph: when a new source->target pair is
-- observed, a CALLS or DEPENDS_ON edge is created/updated in graph_edges.
-- This is the mechanism for eBPF-based topology inference without
-- instrumentation.

CREATE INDEX idx_flows_source_node ON network_flows(source_node_id, timestamp);
CREATE INDEX idx_flows_target_node ON network_flows(target_node_id, timestamp);
```

## RED Metrics Materialized View

```sql
-- ============================================================
-- PRE-AGGREGATED RED METRICS (materialized view)
-- ============================================================

CREATE TABLE red_metrics_1m (
    tenant_id       UUID NOT NULL,
    service_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    operation_name  TEXT NOT NULL,
    bucket_start    TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    duration_sum_ns BIGINT NOT NULL DEFAULT 0,
    duration_min_ns BIGINT,
    duration_max_ns BIGINT,
    PRIMARY KEY (service_node_id, operation_name, bucket_start)
) PARTITION BY RANGE (bucket_start);

CREATE TABLE red_metrics_1m_2026_05 PARTITION OF red_metrics_1m
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Populated by a background job or trigger that aggregates spans
-- into 1-minute buckets grouped by service_node_id and operation.

CREATE INDEX idx_red_metrics_tenant ON red_metrics_1m(tenant_id, bucket_start);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 3 | tenants, users, api_keys |
| Property Graph | 3 | graph_nodes, graph_edges, graph_snapshots |
| Trace Spans | 1 | spans (JSONB attributes — simpler than normalized) |
| Metrics | 2 | metric_descriptors, metric_data_points |
| Logs | 1 | log_records |
| SLOs & Alerts | 5 | slos, slo_snapshots, notification_channels, alert_rules, alert_rule_channels |
| Incidents | 2 | incidents, incident_events |
| AI & Sampling | 2 | ai_analyses, sampling_rules |
| eBPF Flows | 1 | network_flows |
| RED Aggregates | 1 | red_metrics_1m |
| **Total** | **21** | Plus partition tables (auto-generated) |

---

## Key Design Decisions

1. **Property graph as the topology backbone** — The `graph_nodes` and `graph_edges` tables implement a general-purpose property graph within PostgreSQL. Every service, infrastructure component, SLO, and incident is a node; every relationship (CALLS, DEPENDS_ON, AFFECTS, CAUSED_BY, MONITORS) is an edge. This makes topology queries, blast radius analysis, and causal chain traversal first-class operations rather than expensive post-hoc analytics.

2. **Dual representation for key entities** — Services, SLOs, and incidents exist as both relational records (for CRUD operations with constraints) and graph nodes (for topology queries). The `service_node_id`, `slo_node_id`, and `incident_node_id` foreign keys bridge the two worlds. This is intentional duplication: relational tables enforce business rules; graph nodes enable traversal.

3. **Graph-native incident causation** — When the AI root cause analyzer identifies that "payment-db connection pool exhaustion caused order-service latency spike," this is modeled as a CAUSED_BY edge in the graph. The `proposed_graph_changes` field in `ai_analyses` records what graph mutations the AI suggests, and `graph_changes_applied` tracks whether they've been committed. This creates an auditable, explainable causal model.

4. **Temporal graph snapshots** — The `graph_snapshots` table periodically captures the full graph state (nodes + edges with properties) as JSONB. This enables "time travel" queries: "what did the service topology look like before the deployment at 14:18?" — a capability that most observability tools lack entirely.

5. **Topology-aware sampling** — The `topology_aware` sampling rule type leverages the graph to make intelligent sampling decisions. Services with high fan-out (many downstream CALLS edges) are sampled at higher rates because their spans provide more diagnostic value. This is a unique advantage of the graph-first model.

6. **Topology-anomaly alert rules** — The `topology_anomaly` alert rule type detects graph structural changes: new edges (a service started calling a new dependency), missing edges (a dependency relationship disappeared), or edge property changes (error rate on a CALLS edge exceeded threshold). These structural alerts catch issues invisible to metric-based rules.

7. **eBPF flows feed the graph** — Network flows from eBPF/Hubble are stored in a time-partitioned table with foreign keys to graph nodes. A background process materializes new source->target pairs as graph edges, enabling topology inference for uninstrumented services. The graph grows organically from observed traffic patterns.

8. **JSONB attributes on spans** — Unlike Model 1's normalized approach, this model stores span attributes as a single JSONB column. The rationale: in a graph-first architecture, the most expensive queries are topology traversals (handled by the graph layer), not attribute-filtered span searches. JSONB with GIN indexing provides good-enough attribute search while dramatically simplifying the write path.

9. **Recursive CTEs for graph traversal** — PostgreSQL's `WITH RECURSIVE` enables arbitrary-depth graph traversals (blast radius, causal chains, dependency paths) in pure SQL. While slower than Neo4j's native traversal for deep graphs (>5 hops), it handles the typical observability use case (2-4 hops) efficiently and avoids introducing a separate graph database.

10. **Controlled vocabulary for node and edge labels** — Labels like 'service', 'infrastructure', 'CALLS', 'CAUSED_BY' form a controlled vocabulary that enables type-safe graph queries. This is conceptually aligned with Apache TinkerPop's typed property graph model and ensures that graph traversals produce semantically meaningful results.
