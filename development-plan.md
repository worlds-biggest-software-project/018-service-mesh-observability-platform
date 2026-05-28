# Service Mesh Observability Platform — Development Plan

> Project: Candidate #18 · Plan created: 2026-05-25

---

## Technology Decisions

| Category | Choice | Rationale |
|----------|--------|-----------|
| **Primary Language** | Go 1.23+ | De-facto language for cloud-native infrastructure (Kubernetes, Istio, Prometheus, OpenTelemetry Collector are all Go). Excellent concurrency primitives for high-throughput telemetry ingestion. Strong OTLP/gRPC ecosystem support. |
| **API Framework** | Go `net/http` + `connectrpc/connect-go` | Connect provides gRPC-compatible APIs that also work over HTTP/JSON, matching OTLP's dual-protocol model. No heavy framework overhead. |
| **Configuration DB** | PostgreSQL 16+ | ACID guarantees for tenants, SLOs, alert rules, incidents. Row-level security for multi-tenant isolation. Proven at this layer by SigNoz, Grafana, and every major observability platform. |
| **Telemetry DB** | ClickHouse 24.x+ | Columnar storage with 80-95% compression for spans/metrics/logs. Handles 100K-1M+ inserts/second. Materialized views for pre-aggregated RED metrics. Architecture proven by SigNoz at production scale. Selected as Data Model Suggestion 3 (Hybrid Relational + ClickHouse Columnar). |
| **Message Broker** | Apache Kafka / Redpanda | Decouples ingestion from storage. Buffers burst traffic. Enables fan-out to multiple consumers (ClickHouse writer, RED metric aggregator, AI analysis pipeline, sampling evaluator). |
| **Object Storage** | S3-compatible (MinIO for self-hosted) | Long-term trace archive. Cold-tier storage for traces beyond ClickHouse TTL. Supports data residency requirements. |
| **AI/LLM Runtime** | Bring-your-own via LiteLLM proxy | Supports Claude, GPT-4, Llama, Mistral, and local models. Users configure their own API keys or self-hosted models. No vendor lock-in. |
| **Frontend** | React 19 + TypeScript 5.x + Vite | Topology graph visualization with D3.js/react-force-graph. Dashboard charts with Apache ECharts. Proven stack for complex observability UIs (SigNoz, Grafana). |
| **Container Runtime** | Docker + Kubernetes (Helm charts) | Primary deployment target. Helm chart for production. Docker Compose for development/evaluation. |
| **eBPF Agent** | Go + cilium/ebpf library | Kernel-level network flow capture without sidecars. Uses the same eBPF library as Cilium/Hubble. |
| **Testing** | Go `testing` + testify, Playwright (e2e), k6 (load) | Standard Go test tooling. Playwright for frontend integration tests. k6 for telemetry ingestion load testing. |
| **CI/CD** | GitHub Actions | Standard for open-source Go projects. Matrix builds for Linux amd64/arm64. |

---

## Project Directory Structure

```
service-mesh-observability/
├── cmd/
│   ├── collector/              # OTLP collector/ingestion gateway
│   │   └── main.go
│   ├── api-server/             # REST/Connect API server
│   │   └── main.go
│   ├── query-engine/           # Trace/metric/log query service
│   │   └── main.go
│   ├── ai-analyzer/            # LLM-powered root cause analysis
│   │   └── main.go
│   ├── slo-evaluator/          # SLO computation and breach prediction
│   │   └── main.go
│   ├── topology-builder/       # Service dependency graph builder
│   │   └── main.go
│   ├── sampling-controller/    # Intelligent adaptive sampling
│   │   └── main.go
│   ├── alerter/                # Alert evaluation and notification
│   │   └── main.go
│   └── ebpf-agent/             # eBPF network flow agent (per-node)
│       └── main.go
├── internal/
│   ├── model/                  # Domain types (spans, metrics, logs, SLOs)
│   │   ├── span.go
│   │   ├── metric.go
│   │   ├── log.go
│   │   ├── service.go
│   │   ├── slo.go
│   │   ├── incident.go
│   │   ├── alert.go
│   │   ├── tenant.go
│   │   └── sampling.go
│   ├── ingest/                 # OTLP receiver and Kafka producer
│   │   ├── otlp_receiver.go
│   │   ├── kafka_producer.go
│   │   └── batch.go
│   ├── storage/
│   │   ├── clickhouse/         # ClickHouse read/write for telemetry
│   │   │   ├── spans.go
│   │   │   ├── metrics.go
│   │   │   ├── logs.go
│   │   │   └── flows.go
│   │   └── postgres/           # PostgreSQL read/write for config
│   │       ├── tenants.go
│   │       ├── services.go
│   │       ├── slos.go
│   │       ├── alerts.go
│   │       ├── incidents.go
│   │       └── migrations/
│   │           └── *.sql
│   ├── query/                  # Query engine (trace search, metric aggregation)
│   │   ├── trace_query.go
│   │   ├── metric_query.go
│   │   ├── log_query.go
│   │   └── topology_query.go
│   ├── topology/               # Service dependency graph construction
│   │   ├── builder.go
│   │   ├── dependency.go
│   │   └── ebpf_enricher.go
│   ├── ai/                     # AI analysis pipeline
│   │   ├── root_cause.go
│   │   ├── prediction.go
│   │   ├── sampling.go
│   │   ├── governance.go
│   │   └── llm_client.go
│   ├── slo/                    # SLO evaluation engine
│   │   ├── evaluator.go
│   │   ├── burn_rate.go
│   │   └── predictor.go
│   ├── alerting/               # Alert rule evaluation and dispatch
│   │   ├── evaluator.go
│   │   ├── notifier.go
│   │   └── channels/
│   │       ├── slack.go
│   │       ├── pagerduty.go
│   │       ├── opsgenie.go
│   │       ├── webhook.go
│   │       └── email.go
│   ├── sampling/               # Intelligent trace sampling
│   │   ├── head_sampler.go
│   │   ├── tail_sampler.go
│   │   ├── ai_sampler.go
│   │   └── rule_engine.go
│   ├── ebpf/                   # eBPF programs and agent logic
│   │   ├── programs/
│   │   │   └── network_flow.c
│   │   ├── loader.go
│   │   └── flow_parser.go
│   ├── auth/                   # Authentication, API keys, RBAC
│   │   ├── middleware.go
│   │   ├── apikey.go
│   │   └── rbac.go
│   └── config/                 # Configuration loading
│       └── config.go
├── proto/                      # Protocol Buffer definitions
│   ├── api/v1/                 # Public API protos (Connect-RPC)
│   │   ├── trace_service.proto
│   │   ├── metric_service.proto
│   │   ├── log_service.proto
│   │   ├── topology_service.proto
│   │   ├── slo_service.proto
│   │   ├── alert_service.proto
│   │   ├── incident_service.proto
│   │   └── tenant_service.proto
│   └── internal/v1/            # Internal service-to-service protos
│       ├── sampling.proto
│       └── ai_analysis.proto
├── web/                        # React frontend
│   ├── src/
│   │   ├── components/
│   │   │   ├── topology/       # Service dependency graph visualization
│   │   │   ├── traces/         # Trace waterfall view
│   │   │   ├── metrics/        # RED metric dashboards
│   │   │   ├── logs/           # Log search and viewer
│   │   │   ├── slos/           # SLO dashboard and error budgets
│   │   │   ├── incidents/      # Incident timeline and AI summary
│   │   │   ├── alerts/         # Alert rule management
│   │   │   └── shared/         # Shared UI components
│   │   ├── hooks/
│   │   ├── api/                # Generated API client from protos
│   │   ├── stores/             # Zustand state management
│   │   └── App.tsx
│   ├── package.json
│   └── vite.config.ts
├── deploy/
│   ├── docker-compose.yml      # Development/evaluation setup
│   ├── helm/                   # Production Kubernetes deployment
│   │   └── service-mesh-obs/
│   │       ├── Chart.yaml
│   │       ├── values.yaml
│   │       └── templates/
│   └── terraform/              # Infrastructure provisioning
├── scripts/
│   ├── migrate.sh              # Database migration runner
│   ├── generate-proto.sh       # Proto code generation
│   └── seed-demo-data.sh       # Demo data for development
├── test/
│   ├── integration/            # Integration tests (Docker-based)
│   ├── e2e/                    # Playwright end-to-end tests
│   └── load/                   # k6 load test scripts
├── docs/
│   ├── architecture.md
│   ├── api-reference.md
│   └── deployment-guide.md
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
├── docker-compose.dev.yml
└── README.md
```

---

## Phase 1: Foundation — Data Model, Ingestion Pipeline, and Storage

**Purpose:** Establish the core data infrastructure: PostgreSQL schema for configuration, ClickHouse schema for telemetry, OTLP ingestion via Kafka, and basic span/metric/log storage. This phase produces a system that can receive OpenTelemetry data and store it durably, with no UI.

### Task 1.1: PostgreSQL Schema and Migration Infrastructure

**What:** Create the PostgreSQL configuration database with tenant, user, API key, service registry, and migration tooling.

**Design:**

```go
// internal/model/tenant.go
package model

import (
    "time"
    "github.com/google/uuid"
)

type Plan string
const (
    PlanFree       Plan = "free"
    PlanTeam       Plan = "team"
    PlanEnterprise Plan = "enterprise"
)

type Tenant struct {
    ID            uuid.UUID `db:"id"`
    Name          string    `db:"name"`
    Slug          string    `db:"slug"`
    Plan          Plan      `db:"plan"`
    RetentionDays int       `db:"retention_days"`
    MaxSpansPerSec int      `db:"max_spans_per_sec"`
    CreatedAt     time.Time `db:"created_at"`
    UpdatedAt     time.Time `db:"updated_at"`
}

type UserRole string
const (
    RoleOwner  UserRole = "owner"
    RoleAdmin  UserRole = "admin"
    RoleEditor UserRole = "editor"
    RoleViewer UserRole = "viewer"
)

type User struct {
    ID          uuid.UUID `db:"id"`
    TenantID    uuid.UUID `db:"tenant_id"`
    Email       string    `db:"email"`
    DisplayName string    `db:"display_name"`
    Role        UserRole  `db:"role"`
    CreatedAt   time.Time `db:"created_at"`
}

type APIKey struct {
    ID        uuid.UUID  `db:"id"`
    TenantID  uuid.UUID  `db:"tenant_id"`
    Name      string     `db:"name"`
    KeyHash   string     `db:"key_hash"`
    Scopes    []string   `db:"scopes"`
    ExpiresAt *time.Time `db:"expires_at"`
    CreatedAt time.Time  `db:"created_at"`
}
```

```sql
-- internal/storage/postgres/migrations/001_tenants_and_users.sql

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

-- Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE services ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON api_keys
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON services
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE INDEX idx_services_tenant ON services(tenant_id, name);
```

```go
// internal/storage/postgres/tenants.go
package postgres

import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
)

type TenantStore struct {
    pool *pgxpool.Pool
}

func NewTenantStore(pool *pgxpool.Pool) *TenantStore {
    return &TenantStore{pool: pool}
}

func (s *TenantStore) Create(ctx context.Context, t *model.Tenant) error { ... }
func (s *TenantStore) GetBySlug(ctx context.Context, slug string) (*model.Tenant, error) { ... }
func (s *TenantStore) SetTenantContext(ctx context.Context, tenantID uuid.UUID) error {
    _, err := s.pool.Exec(ctx,
        "SET LOCAL app.current_tenant_id = $1", tenantID.String())
    return err
}
```

**Testing:**

- Unit: Validate `Tenant` struct serialization round-trip with all `Plan` values
- Unit: Verify `CREATE TABLE` DDL executes cleanly against a fresh PostgreSQL instance (testcontainers)
- Integration: Create tenant, create user under tenant, verify RLS blocks cross-tenant user reads
- Integration: Create API key, verify `key_hash` uniqueness constraint on duplicate insert
- Integration: Verify migration runner applies all `*.sql` files in order, is idempotent on re-run

---

### Task 1.2: ClickHouse Telemetry Schema

**What:** Create the ClickHouse schema for spans, metrics, logs, and network flows with materialized views for RED metric pre-aggregation.

**Design:**

```sql
-- Span index table (modeled after SigNoz signoz_index_v3)
CREATE TABLE spans
(
    `ts_bucket_start`       UInt64 CODEC(DoubleDelta, LZ4),
    `resource_fingerprint`  String CODEC(ZSTD(1)),
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `trace_id`              FixedString(32) CODEC(ZSTD(1)),
    `span_id`               String CODEC(ZSTD(1)),
    `parent_span_id`        String CODEC(ZSTD(1)),
    `trace_state`           String CODEC(ZSTD(1)),
    `name`                  LowCardinality(String) CODEC(ZSTD(1)),
    `kind`                  Int8 CODEC(T64, ZSTD(1)),
    `kind_string`           LowCardinality(String) CODEC(ZSTD(1)),
    `duration_nano`         UInt64 CODEC(T64, ZSTD(1)),
    `status_code`           Int16 CODEC(T64, ZSTD(1)),
    `status_message`        String CODEC(ZSTD(1)),
    `has_error`             Bool CODEC(ZSTD(1)),
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),
    `http_method`           LowCardinality(String) CODEC(ZSTD(1)),
    `http_url`              String CODEC(ZSTD(2)),
    `http_route`            LowCardinality(String) CODEC(ZSTD(1)),
    `http_status_code`      Int16 CODEC(T64, ZSTD(1)),
    `rpc_method`            LowCardinality(String) CODEC(ZSTD(1)),
    `rpc_system`            LowCardinality(String) CODEC(ZSTD(1)),
    `db_system`             LowCardinality(String) CODEC(ZSTD(1)),
    `db_name`               LowCardinality(String) CODEC(ZSTD(1)),
    `db_operation`          LowCardinality(String) CODEC(ZSTD(1)),
    `messaging_system`      LowCardinality(String) CODEC(ZSTD(1)),
    `messaging_operation`   LowCardinality(String) CODEC(ZSTD(1)),
    `attributes_string`     Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    `attributes_number`     Map(LowCardinality(String), Float64) CODEC(ZSTD(1)),
    `attributes_bool`       Map(LowCardinality(String), Bool) CODEC(ZSTD(1)),
    `resources_string`      Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    `event_names`           Array(String) CODEC(ZSTD(2)),
    `event_times`           Array(DateTime64(9)) CODEC(ZSTD(1)),
    `event_attributes`      Array(Map(LowCardinality(String), String)) CODEC(ZSTD(1)),
    `link_trace_ids`        Array(FixedString(32)) CODEC(ZSTD(1)),
    `link_span_ids`         Array(String) CODEC(ZSTD(1)),
    `sampling_decision`     LowCardinality(String) CODEC(ZSTD(1)),
    `novelty_score`         Float32 CODEC(ZSTD(1)),
    `scope_name`            LowCardinality(String) CODEC(ZSTD(1)),
    `scope_version`         LowCardinality(String) CODEC(ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, ts_bucket_start, resource_fingerprint, has_error, name, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(30)
SETTINGS index_granularity = 8192;

ALTER TABLE spans ADD INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE spans ADD INDEX idx_span_id span_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE spans ADD INDEX idx_duration duration_nano TYPE minmax GRANULARITY 1;

-- RED metrics materialized view (1-minute buckets)
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
    `duration_quantiles` AggregateFunction(quantiles(0.5, 0.75, 0.9, 0.95, 0.99), UInt64)
)
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(bucket_start)
ORDER BY (tenant_id, service_name, operation_name, span_kind, bucket_start)
TTL bucket_start + toIntervalDay(90)
SETTINGS index_granularity = 8192;

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
    quantilesState(0.5, 0.75, 0.9, 0.95, 0.99)(duration_nano) AS duration_quantiles
FROM spans
WHERE kind IN (2, 4)
GROUP BY tenant_id, service_name, operation_name, span_kind, bucket_start;
```

```go
// internal/storage/clickhouse/spans.go
package clickhouse

import (
    "context"
    "github.com/ClickHouse/clickhouse-go/v2"
)

type SpanWriter struct {
    conn clickhouse.Conn
}

func NewSpanWriter(conn clickhouse.Conn) *SpanWriter {
    return &SpanWriter{conn: conn}
}

// WriteBatch inserts a batch of spans into ClickHouse.
// Uses async insert for throughput. Batch size target: 10,000 spans.
func (w *SpanWriter) WriteBatch(ctx context.Context, spans []model.Span) error { ... }

type SpanReader struct {
    conn clickhouse.Conn
}

// GetTraceByID reconstructs a full trace from span data.
func (r *SpanReader) GetTraceByID(ctx context.Context, tenantID, traceID string) ([]model.Span, error) { ... }
```

**Testing:**

- Unit: Verify ClickHouse DDL creates tables without error against a test ClickHouse instance (testcontainers)
- Unit: Insert 1,000 spans, verify `red_metrics_1m` materialized view populates with correct aggregate counts
- Integration: Insert spans with `has_error=true`, verify bloom filter index enables fast trace_id lookup
- Integration: Verify TTL configuration by inserting data with past timestamps and running `OPTIMIZE TABLE`
- Load: Insert 50,000 spans/second for 60 seconds, verify no data loss and query latency remains < 500ms

---

### Task 1.3: OTLP Ingestion Receiver

**What:** Build an OTLP-compatible gRPC and HTTP receiver that accepts spans, metrics, and logs following the OpenTelemetry protocol specification.

**Design:**

```go
// internal/ingest/otlp_receiver.go
package ingest

import (
    "context"
    colpb "go.opentelemetry.io/proto/otlp/collector/trace/v1"
    tracepb "go.opentelemetry.io/proto/otlp/trace/v1"
)

// OTLPReceiver implements the OTLP gRPC and HTTP endpoints.
// It validates incoming data, extracts tenant_id from request metadata,
// and forwards to the Kafka producer for async processing.
type OTLPReceiver struct {
    producer  *KafkaProducer
    validator *SpanValidator
    config    ReceiverConfig
}

type ReceiverConfig struct {
    GRPCAddr       string        // e.g., ":4317"
    HTTPAddr       string        // e.g., ":4318"
    MaxBatchSize   int           // max spans per request (default: 10000)
    MaxRequestSize int64         // max request body bytes (default: 4MB)
    TenantHeader   string        // HTTP header for tenant ID (default: "X-Tenant-ID")
}

// Export handles the OTLP ExportTraceServiceRequest.
// 1. Extracts tenant_id from gRPC metadata or HTTP header
// 2. Validates span structure (trace_id format, required fields)
// 3. Produces validated spans to Kafka topic "otlp.spans.v1"
// 4. Returns ExportTraceServiceResponse with partial success counts
func (r *OTLPReceiver) Export(
    ctx context.Context,
    req *colpb.ExportTraceServiceRequest,
) (*colpb.ExportTraceServiceResponse, error) { ... }

// SpanValidator checks OTLP span conformance.
type SpanValidator struct{}

func (v *SpanValidator) ValidateSpan(span *tracepb.Span) error {
    // - trace_id must be 16 bytes (128-bit, W3C Trace Context)
    // - span_id must be 8 bytes (64-bit)
    // - name must be non-empty
    // - start_time_unix_nano must be > 0
    // - end_time_unix_nano must be >= start_time_unix_nano
    // - kind must be 0-5
    // - status.code must be 0-2
    return nil
}
```

```go
// internal/ingest/kafka_producer.go
package ingest

type KafkaProducer struct {
    writer *kafka.Writer
    config KafkaConfig
}

type KafkaConfig struct {
    Brokers       []string
    SpansTopic    string // "otlp.spans.v1"
    MetricsTopic  string // "otlp.metrics.v1"
    LogsTopic     string // "otlp.logs.v1"
    BatchSize     int    // 1000
    BatchTimeout  time.Duration // 100ms
    Compression   string // "zstd"
}

// ProduceSpans serializes spans to protobuf and writes to Kafka.
// Key: tenant_id (for partition co-locality of same-tenant data).
func (p *KafkaProducer) ProduceSpans(ctx context.Context, tenantID string, spans []*tracepb.ResourceSpans) error { ... }
```

**Testing:**

- Unit: `SpanValidator` rejects span with empty trace_id, zero start_time, invalid kind value
- Unit: `SpanValidator` accepts a well-formed OTLP span with all optional fields populated
- Integration: Send ExportTraceServiceRequest via gRPC to receiver, verify message appears in Kafka `otlp.spans.v1` topic
- Integration: Send trace data via HTTP POST to `/v1/traces` with `X-Tenant-ID` header, verify Kafka receipt
- Integration: Send oversized request (>4MB), verify HTTP 413 / gRPC RESOURCE_EXHAUSTED response
- Integration: Send request without tenant header, verify HTTP 401 / gRPC UNAUTHENTICATED response

---

### Task 1.4: Kafka-to-ClickHouse Span Writer

**What:** Build a Kafka consumer that reads spans from the `otlp.spans.v1` topic, transforms OTLP protobuf into ClickHouse row format, and batch-inserts into the `spans` table.

**Design:**

```go
// internal/ingest/span_consumer.go
package ingest

type SpanConsumer struct {
    reader       *kafka.Reader
    spanWriter   *clickhouse.SpanWriter
    serviceStore *postgres.ServiceStore
    config       ConsumerConfig
}

type ConsumerConfig struct {
    GroupID          string        // "span-writer"
    BatchSize        int           // 5000 spans per batch insert
    BatchTimeout     time.Duration // 500ms max wait before flushing
    WorkerCount      int           // 4 parallel workers
    CommitInterval   time.Duration // 1s
}

// Run starts consuming from Kafka and writing to ClickHouse.
// Processing pipeline per message:
// 1. Deserialize protobuf ResourceSpans
// 2. Extract resource attributes, compute resource_fingerprint (SHA256 of sorted keys)
// 3. Extract hot attributes (http_method, http_url, db_system, etc.) into top-level fields
// 4. Upsert service name into PostgreSQL service registry (debounced, 1/minute per service)
// 5. Accumulate into batch buffer
// 6. Flush batch to ClickHouse when buffer reaches BatchSize or BatchTimeout
func (c *SpanConsumer) Run(ctx context.Context) error { ... }

// transformSpan converts OTLP Span proto to ClickHouse row struct.
func transformSpan(
    tenantID string,
    resource *tracepb.Resource,
    fingerprint string,
    span *tracepb.Span,
) *clickhouse.SpanRow { ... }
```

```go
// internal/storage/clickhouse/spans.go (SpanRow struct)
type SpanRow struct {
    TSBucketStart      uint64            `ch:"ts_bucket_start"`
    ResourceFingerprint string           `ch:"resource_fingerprint"`
    Timestamp          time.Time         `ch:"timestamp"`
    TraceID            string            `ch:"trace_id"`
    SpanID             string            `ch:"span_id"`
    ParentSpanID       string            `ch:"parent_span_id"`
    Name               string            `ch:"name"`
    Kind               int8              `ch:"kind"`
    KindString         string            `ch:"kind_string"`
    DurationNano       uint64            `ch:"duration_nano"`
    StatusCode         int16             `ch:"status_code"`
    StatusMessage      string            `ch:"status_message"`
    HasError           bool              `ch:"has_error"`
    TenantID           string            `ch:"tenant_id"`
    HTTPMethod         string            `ch:"http_method"`
    HTTPURL            string            `ch:"http_url"`
    HTTPRoute          string            `ch:"http_route"`
    HTTPStatusCode     int16             `ch:"http_status_code"`
    RPCMethod          string            `ch:"rpc_method"`
    RPCSystem          string            `ch:"rpc_system"`
    DBSystem           string            `ch:"db_system"`
    DBName             string            `ch:"db_name"`
    DBOperation        string            `ch:"db_operation"`
    AttributesString   map[string]string `ch:"attributes_string"`
    AttributesNumber   map[string]float64 `ch:"attributes_number"`
    ResourcesString    map[string]string `ch:"resources_string"`
    SamplingDecision   string            `ch:"sampling_decision"`
    NoveltyScore       float32           `ch:"novelty_score"`
}
```

**Testing:**

- Unit: `transformSpan` correctly extracts `http_method`, `db_system` from OTLP attribute arrays
- Unit: `transformSpan` computes `ts_bucket_start` as floor(unix_timestamp / 1800) * 1800
- Unit: `transformSpan` sets `has_error = true` when `status_code = 2`
- Integration: Produce 10,000 spans to Kafka, verify all 10,000 are inserted into ClickHouse within 5 seconds
- Integration: Produce spans for 3 services, verify PostgreSQL `services` table contains all 3
- Integration: Kill consumer mid-batch, restart, verify no duplicate spans (idempotent via Kafka offset commit)

---

### Task 1.5: Docker Compose Development Environment

**What:** Create a Docker Compose configuration that runs the full development stack: PostgreSQL, ClickHouse, Kafka (Redpanda), and the collector service.

**Design:**

```yaml
# docker-compose.dev.yml
version: "3.8"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: meshobs
      POSTGRES_USER: meshobs
      POSTGRES_PASSWORD: dev-password
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U meshobs"]
      interval: 5s
      timeout: 3s
      retries: 5

  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    ports:
      - "8123:8123"   # HTTP
      - "9000:9000"   # Native
    volumes:
      - ch_data:/var/lib/clickhouse
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    healthcheck:
      test: ["CMD", "clickhouse-client", "--query", "SELECT 1"]
      interval: 5s
      timeout: 3s
      retries: 5

  redpanda:
    image: redpandadata/redpanda:v24.1
    command:
      - redpanda start
      - --mode dev-container
      - --smp 1
      - --memory 512M
    ports:
      - "9092:9092"   # Kafka API
      - "8082:8082"   # Schema Registry
      - "9644:9644"   # Admin API
    healthcheck:
      test: ["CMD", "rpk", "cluster", "health"]
      interval: 5s
      timeout: 3s
      retries: 5

  collector:
    build:
      context: .
      dockerfile: Dockerfile
      target: collector
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    environment:
      POSTGRES_DSN: "postgres://meshobs:dev-password@postgres:5432/meshobs?sslmode=disable"
      CLICKHOUSE_DSN: "clickhouse://clickhouse:9000/default"
      KAFKA_BROKERS: "redpanda:9092"
    depends_on:
      postgres: { condition: service_healthy }
      clickhouse: { condition: service_healthy }
      redpanda: { condition: service_healthy }

volumes:
  pg_data:
  ch_data:
```

```go
// internal/config/config.go
package config

type Config struct {
    Collector  CollectorConfig
    Postgres   PostgresConfig
    ClickHouse ClickHouseConfig
    Kafka      KafkaConfig
}

type CollectorConfig struct {
    GRPCAddr       string `env:"COLLECTOR_GRPC_ADDR" default:":4317"`
    HTTPAddr       string `env:"COLLECTOR_HTTP_ADDR" default:":4318"`
    TenantHeader   string `env:"COLLECTOR_TENANT_HEADER" default:"X-Tenant-ID"`
    MaxBatchSize   int    `env:"COLLECTOR_MAX_BATCH_SIZE" default:"10000"`
}

type PostgresConfig struct {
    DSN            string `env:"POSTGRES_DSN" required:"true"`
    MaxConns       int    `env:"POSTGRES_MAX_CONNS" default:"20"`
    MigrationsPath string `env:"POSTGRES_MIGRATIONS" default:"internal/storage/postgres/migrations"`
}

type ClickHouseConfig struct {
    DSN            string `env:"CLICKHOUSE_DSN" required:"true"`
    MaxConns       int    `env:"CLICKHOUSE_MAX_CONNS" default:"10"`
    AsyncInsert    bool   `env:"CLICKHOUSE_ASYNC_INSERT" default:"true"`
}

// Load reads config from environment variables with defaults.
func Load() (*Config, error) { ... }
```

**Testing:**

- Integration: `docker compose up -d` starts all services within 30 seconds, all healthchecks pass
- Integration: Send OTLP spans to `localhost:4317`, query ClickHouse at `localhost:8123` to verify data arrived
- Integration: `docker compose down -v && docker compose up -d` starts from clean state without errors
- Unit: `config.Load()` uses defaults when env vars are unset, returns error when required vars are missing

---

## Phase 2: Query Engine and Trace Reconstruction

**Purpose:** Build the query layer that reads from ClickHouse to reconstruct traces, search spans, and render RED metrics. This phase produces APIs that a frontend or Grafana data source plugin can consume.

### Task 2.1: Trace Query Service

**What:** Build a query service that retrieves traces by ID, searches spans by attribute filters, and computes trace-level aggregates.

**Design:**

```go
// internal/query/trace_query.go
package query

type TraceQueryService struct {
    spanReader *clickhouse.SpanReader
}

// GetTrace retrieves all spans for a trace, ordered by start time.
// Returns spans grouped by service for waterfall rendering.
type TraceResult struct {
    TraceID    string        `json:"trace_id"`
    RootSpan   *model.Span   `json:"root_span"`
    Spans      []model.Span  `json:"spans"`
    SpanCount  int           `json:"span_count"`
    ServiceCount int         `json:"service_count"`
    DurationNs uint64        `json:"duration_ns"`
    HasError   bool          `json:"has_error"`
}

func (s *TraceQueryService) GetTrace(ctx context.Context, tenantID, traceID string) (*TraceResult, error) { ... }

// SearchSpans finds spans matching filter criteria.
type SpanSearchRequest struct {
    TenantID      string            `json:"tenant_id"`
    ServiceName   string            `json:"service_name,omitempty"`
    OperationName string            `json:"operation_name,omitempty"`
    MinDurationMs int64             `json:"min_duration_ms,omitempty"`
    MaxDurationMs int64             `json:"max_duration_ms,omitempty"`
    HasError      *bool             `json:"has_error,omitempty"`
    Attributes    map[string]string `json:"attributes,omitempty"`
    StartTime     time.Time         `json:"start_time"`
    EndTime       time.Time         `json:"end_time"`
    Limit         int               `json:"limit"`      // max 1000
    Offset        int               `json:"offset"`
    OrderBy       string            `json:"order_by"`   // "timestamp", "duration"
    OrderDir      string            `json:"order_dir"`  // "asc", "desc"
}

type SpanSearchResult struct {
    Spans      []model.Span `json:"spans"`
    TotalCount int64        `json:"total_count"`
    HasMore    bool         `json:"has_more"`
}

func (s *TraceQueryService) SearchSpans(ctx context.Context, req *SpanSearchRequest) (*SpanSearchResult, error) { ... }
```

```sql
-- Generated ClickHouse query for SearchSpans:
SELECT
    trace_id, span_id, parent_span_id, name, kind_string,
    duration_nano, status_code, status_message, has_error,
    attributes_string, resources_string,
    resources_string['service.name'] AS service_name,
    timestamp
FROM spans
WHERE tenant_id = {tenant_id:String}
  AND timestamp >= {start_time:DateTime64(9)}
  AND timestamp <= {end_time:DateTime64(9)}
  AND ts_bucket_start >= {bucket_start:UInt64} - 1800
  AND ts_bucket_start <= {bucket_end:UInt64} + 1800
  -- Optional filters (conditionally appended):
  AND resources_string['service.name'] = {service_name:String}
  AND name = {operation_name:String}
  AND duration_nano >= {min_duration_ns:UInt64}
  AND has_error = {has_error:Bool}
  AND attributes_string[{attr_key:String}] = {attr_value:String}
ORDER BY {order_by} {order_dir}
LIMIT {limit:UInt32}
OFFSET {offset:UInt32}
```

**Testing:**

- Unit: `SearchSpans` with service_name filter generates correct ClickHouse SQL with `resources_string['service.name']` predicate
- Unit: `SearchSpans` with `min_duration_ms=100` converts to nanoseconds (100000000) in the query
- Integration: Insert 5 traces (20 spans each), retrieve by trace_id, verify root_span identified correctly
- Integration: Insert spans across 3 services, search by service_name, verify only matching spans returned
- Integration: Insert 100 spans, search with `limit=10`, verify `has_more=true` and `total_count=100`
- Integration: Search with attribute filter `http.status_code=500`, verify only error spans returned

---

### Task 2.2: RED Metric Query Service

**What:** Build a service that queries pre-aggregated RED metrics from the ClickHouse materialized view for dashboard rendering.

**Design:**

```go
// internal/query/metric_query.go
package query

type REDMetricService struct {
    conn clickhouse.Conn
}

type REDMetricRequest struct {
    TenantID      string    `json:"tenant_id"`
    ServiceName   string    `json:"service_name"`
    OperationName string    `json:"operation_name,omitempty"` // empty = all operations
    StartTime     time.Time `json:"start_time"`
    EndTime       time.Time `json:"end_time"`
    StepSeconds   int       `json:"step_seconds"` // 60 (1m), 300 (5m), 3600 (1h)
}

type REDMetricPoint struct {
    Timestamp    time.Time `json:"timestamp"`
    RequestCount uint64    `json:"request_count"`
    ErrorCount   uint64    `json:"error_count"`
    ErrorRate    float64   `json:"error_rate"`    // error_count / request_count
    P50LatencyMs float64   `json:"p50_latency_ms"`
    P95LatencyMs float64   `json:"p95_latency_ms"`
    P99LatencyMs float64   `json:"p99_latency_ms"`
}

type REDMetricResponse struct {
    ServiceName string           `json:"service_name"`
    DataPoints  []REDMetricPoint `json:"data_points"`
}

func (s *REDMetricService) GetREDMetrics(ctx context.Context, req *REDMetricRequest) (*REDMetricResponse, error) { ... }

// ServiceOverview returns RED metrics for all services in a tenant.
type ServiceOverviewEntry struct {
    ServiceName  string  `json:"service_name"`
    RequestRate  float64 `json:"request_rate_per_sec"`
    ErrorRate    float64 `json:"error_rate"`
    P99LatencyMs float64 `json:"p99_latency_ms"`
}

func (s *REDMetricService) GetServiceOverview(ctx context.Context, tenantID string, duration time.Duration) ([]ServiceOverviewEntry, error) { ... }
```

**Testing:**

- Unit: `StepSeconds=3600` queries `red_metrics_1h` table; `StepSeconds=60` queries `red_metrics_1m`
- Integration: Insert 10,000 spans (5% with errors), query RED metrics, verify error_rate ~ 0.05
- Integration: Insert spans across 1-hour window, query with `StepSeconds=60`, verify 60 data points returned
- Integration: `GetServiceOverview` lists all services with non-zero traffic in the requested window

---

### Task 2.3: API Server (Connect-RPC)

**What:** Build the public API server using Connect-RPC that exposes trace, metric, and configuration endpoints over both gRPC and HTTP/JSON.

**Design:**

```protobuf
// proto/api/v1/trace_service.proto
syntax = "proto3";
package api.v1;

service TraceService {
  rpc GetTrace(GetTraceRequest) returns (GetTraceResponse);
  rpc SearchSpans(SearchSpansRequest) returns (SearchSpansResponse);
  rpc GetServiceOverview(GetServiceOverviewRequest) returns (GetServiceOverviewResponse);
  rpc GetREDMetrics(GetREDMetricsRequest) returns (GetREDMetricsResponse);
}

message GetTraceRequest {
  string trace_id = 1;
}

message GetTraceResponse {
  string trace_id = 1;
  Span root_span = 2;
  repeated Span spans = 3;
  int32 span_count = 4;
  int32 service_count = 5;
  uint64 duration_ns = 6;
  bool has_error = 7;
}

message Span {
  string trace_id = 1;
  string span_id = 2;
  string parent_span_id = 3;
  string name = 4;
  string kind = 5;
  uint64 start_time_unix_nano = 6;
  uint64 end_time_unix_nano = 7;
  uint64 duration_ns = 8;
  int32 status_code = 9;
  string status_message = 10;
  bool has_error = 11;
  string service_name = 12;
  map<string, string> attributes = 13;
  map<string, string> resource_attributes = 14;
}

message SearchSpansRequest {
  string service_name = 1;
  string operation_name = 2;
  int64 min_duration_ms = 3;
  int64 max_duration_ms = 4;
  optional bool has_error = 5;
  map<string, string> attributes = 6;
  int64 start_time_unix_ms = 7;
  int64 end_time_unix_ms = 8;
  int32 limit = 9;
  int32 offset = 10;
  string order_by = 11;
}

message SearchSpansResponse {
  repeated Span spans = 1;
  int64 total_count = 2;
  bool has_more = 3;
}
```

```go
// cmd/api-server/main.go
package main

func main() {
    cfg := config.Load()

    // Connect handler setup
    mux := http.NewServeMux()

    traceHandler := connectHandler(traceService)
    mux.Handle(tracev1connect.NewTraceServiceHandler(traceHandler))

    tenantHandler := connectHandler(tenantService)
    mux.Handle(tenantv1connect.NewTenantServiceHandler(tenantHandler))

    // Auth middleware: extract tenant from API key or JWT
    authed := auth.Middleware(mux, apiKeyStore)

    // CORS for browser clients
    corsHandler := cors.Handler(authed)

    server := &http.Server{
        Addr:    cfg.API.Addr,    // ":8080"
        Handler: corsHandler,
    }
    server.ListenAndServe()
}
```

**Testing:**

- Unit: Proto generation compiles without errors; Go, TypeScript clients are generated
- Integration: Call `GetTrace` via gRPC with valid trace_id, verify response contains expected spans
- Integration: Call `SearchSpans` via HTTP POST `/api.v1.TraceService/SearchSpans` with JSON body, verify JSON response
- Integration: Call API without API key header, verify UNAUTHENTICATED error
- Integration: Call API with API key for tenant A, request trace belonging to tenant B, verify NOT_FOUND (RLS)

---

## Phase 3: Service Topology and Dependency Graph

**Purpose:** Build the service dependency topology from trace data and eBPF network flows. Produce an API that returns the live service graph with health status on edges.

### Task 3.1: Trace-Derived Topology Builder

**What:** Build a background service that processes CLIENT spans from ClickHouse to discover and maintain service dependency relationships in the PostgreSQL service registry.

**Design:**

```go
// internal/topology/builder.go
package topology

type TopologyBuilder struct {
    spanReader   *clickhouse.SpanReader
    serviceStore *postgres.ServiceStore
    depStore     *postgres.DependencyStore
    interval     time.Duration // 1 minute
}

type ServiceDependency struct {
    TenantID      string `db:"tenant_id"`
    SourceService string `db:"source_service"`
    TargetService string `db:"target_service"`
    Protocol      string `db:"protocol"`     // "http", "grpc", "kafka", "redis"
    CallCount1h   int64  `db:"call_count_1h"`
    ErrorCount1h  int64  `db:"error_count_1h"`
    ErrorRate1h   float64 `db:"error_rate_1h"`
    P50LatencyMs  float64 `db:"p50_latency_ms"`
    P99LatencyMs  float64 `db:"p99_latency_ms"`
    DiscoveredVia string `db:"discovered_via"` // "trace" or "ebpf"
    FirstSeenAt   time.Time `db:"first_seen_at"`
    LastSeenAt    time.Time `db:"last_seen_at"`
}

// Run periodically queries the ClickHouse service_dependencies_agg
// materialized view and upserts dependency edges into PostgreSQL.
func (b *TopologyBuilder) Run(ctx context.Context) error {
    ticker := time.NewTicker(b.interval)
    for {
        select {
        case <-ctx.Done():
            return nil
        case <-ticker.C:
            b.buildTopology(ctx)
        }
    }
}

func (b *TopologyBuilder) buildTopology(ctx context.Context) error {
    // 1. Query ClickHouse service_dependencies_agg for last 1 hour
    // 2. For each (source, target, protocol) tuple:
    //    a. Upsert source and target into services table
    //    b. Upsert dependency edge with call/error counts
    //    c. Update last_seen_at
    // 3. Mark dependencies not seen in 24h as inactive
    return nil
}
```

```sql
-- internal/storage/postgres/migrations/002_service_dependencies.sql

CREATE TABLE service_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_service  TEXT NOT NULL,
    target_service  TEXT NOT NULL,
    protocol        TEXT,
    call_count_1h   BIGINT NOT NULL DEFAULT 0,
    error_count_1h  BIGINT NOT NULL DEFAULT 0,
    error_rate_1h   NUMERIC(5,4) NOT NULL DEFAULT 0,
    p50_latency_ms  NUMERIC(10,2),
    p99_latency_ms  NUMERIC(10,2),
    discovered_via  TEXT NOT NULL DEFAULT 'trace'
                    CHECK (discovered_via IN ('trace', 'ebpf', 'config', 'manual')),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, source_service, target_service, protocol)
);

ALTER TABLE service_dependencies ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON service_dependencies
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE INDEX idx_deps_tenant ON service_dependencies(tenant_id);
CREATE INDEX idx_deps_source ON service_dependencies(tenant_id, source_service);
CREATE INDEX idx_deps_target ON service_dependencies(tenant_id, target_service);
```

**Testing:**

- Unit: `buildTopology` correctly merges CLIENT spans into dependency edges grouped by protocol
- Integration: Insert spans with `kind=CLIENT` and `peer.service` attribute, verify dependency edge created
- Integration: Send spans for A->B (http) and A->C (grpc), verify 2 dependency edges with correct protocols
- Integration: Stop sending spans for A->C, wait 24h (simulated), verify edge marked `is_active=false`
- Integration: Verify `discovered_via='trace'` is set for trace-derived dependencies

---

### Task 3.2: Topology API

**What:** Build API endpoints that return the service dependency graph with health metrics on nodes and edges, suitable for frontend visualization.

**Design:**

```protobuf
// proto/api/v1/topology_service.proto
syntax = "proto3";
package api.v1;

service TopologyService {
  rpc GetTopology(GetTopologyRequest) returns (GetTopologyResponse);
  rpc GetServiceDetail(GetServiceDetailRequest) returns (GetServiceDetailResponse);
}

message GetTopologyRequest {
  string environment = 1;  // "production", "staging", or empty for all
  string namespace = 2;    // filter by k8s namespace
  int64 since_minutes = 3; // default 60
}

message GetTopologyResponse {
  repeated TopologyNode nodes = 1;
  repeated TopologyEdge edges = 2;
}

message TopologyNode {
  string service_name = 1;
  string namespace = 2;
  string environment = 3;
  double request_rate = 4;      // requests/sec
  double error_rate = 5;        // 0.0-1.0
  double p99_latency_ms = 6;
  string health_status = 7;     // "healthy", "degraded", "critical"
}

message TopologyEdge {
  string source = 1;
  string target = 2;
  string protocol = 3;
  int64 call_count = 4;
  double error_rate = 5;
  double p99_latency_ms = 6;
  string discovered_via = 7;
}
```

```go
// Health status computation
func computeHealthStatus(errorRate float64, p99LatencyMs float64) string {
    if errorRate > 0.05 { return "critical" }
    if errorRate > 0.01 { return "degraded" }
    if p99LatencyMs > 1000 { return "degraded" }
    return "healthy"
}
```

**Testing:**

- Unit: `computeHealthStatus` returns "critical" for error_rate > 5%, "degraded" for > 1%, "healthy" otherwise
- Integration: Insert 3-service topology (A->B->C), call `GetTopology`, verify 3 nodes and 2 edges
- Integration: Insert spans with 10% error rate for service B, verify B's health_status = "critical"
- Integration: Filter by namespace, verify only services in that namespace appear

---

## Phase 4: SLO Engine, Alerting, and Notifications

**Purpose:** Build the SLO evaluation engine with error budget tracking and burn rate computation, the alert rule evaluator, and notification dispatch to Slack/PagerDuty/webhook.

### Task 4.1: SLO CRUD and Evaluation Engine

**What:** Build the SLO management API and a background evaluator that computes SLI values, error budgets, and multi-window burn rates every minute.

**Design:**

```go
// internal/slo/evaluator.go
package slo

type SLOEvaluator struct {
    sloStore      *postgres.SLOStore
    spanReader    *clickhouse.SpanReader
    snapshotStore *postgres.SLOSnapshotStore
    interval      time.Duration // 1 minute
}

type SLOSnapshot struct {
    SLOID               uuid.UUID `db:"slo_id"`
    ComputedAt          time.Time `db:"computed_at"`
    WindowStart         time.Time `db:"window_start"`
    WindowEnd           time.Time `db:"window_end"`
    GoodEvents          int64     `db:"good_events"`
    TotalEvents         int64     `db:"total_events"`
    SLIValue            float64   `db:"sli_value"`          // e.g., 99.95
    ErrorBudgetRemaining float64  `db:"error_budget_remaining"` // e.g., 0.03 (3%)
    BurnRate1h          float64   `db:"burn_rate_1h"`
    BurnRate6h          float64   `db:"burn_rate_6h"`
    BurnRate24h         float64   `db:"burn_rate_24h"`
}

// ComputeBurnRate calculates the burn rate for a given time window.
// burn_rate = (error_rate_in_window / allowed_error_rate)
// A burn rate of 1.0 means the error budget is being consumed at exactly
// the rate that would exhaust it by end of SLO window.
// A burn rate of 14.4 (Google SRE recommendation) means the budget
// will exhaust in ~1 hour for a 30-day window.
func ComputeBurnRate(goodEvents, totalEvents int64, targetPercent float64) float64 {
    if totalEvents == 0 { return 0 }
    actualGoodRate := float64(goodEvents) / float64(totalEvents)
    allowedBadRate := 1.0 - (targetPercent / 100.0)
    actualBadRate := 1.0 - actualGoodRate
    if allowedBadRate == 0 { return 0 }
    return actualBadRate / allowedBadRate
}
```

```sql
-- internal/storage/postgres/migrations/003_slos.sql

CREATE TABLE slos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    service_name    TEXT NOT NULL,
    name            TEXT NOT NULL,
    sli_type        TEXT NOT NULL
                    CHECK (sli_type IN ('availability', 'latency', 'throughput', 'error_rate')),
    target_percent  NUMERIC(6,3) NOT NULL,
    window_type     TEXT NOT NULL DEFAULT 'rolling'
                    CHECK (window_type IN ('rolling', 'calendar')),
    window_days     INT NOT NULL DEFAULT 30,
    sli_query       JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    burn_rate_6h    NUMERIC(8,5),
    burn_rate_24h   NUMERIC(8,5)
);

CREATE INDEX idx_slo_snapshots_slo_time ON slo_snapshots(slo_id, computed_at);
```

**Testing:**

- Unit: `ComputeBurnRate(9990, 10000, 99.9)` returns 10.0 (10x budget consumption rate)
- Unit: `ComputeBurnRate(10000, 10000, 99.9)` returns 0.0 (no errors)
- Unit: `ComputeBurnRate(0, 0, 99.9)` returns 0.0 (no traffic)
- Integration: Create SLO (99.9% availability, 30-day rolling), insert spans with 0.5% error rate, verify SLI = 99.5
- Integration: Verify error_budget_remaining decreases when error rate exceeds target
- Integration: Verify burn_rate_1h calculation matches expected value for known error rate
- Integration: SLO with `sli_type=latency` counts spans with duration < threshold as good events

---

### Task 4.2: Alert Rule Engine and Notification Dispatch

**What:** Build the alert rule evaluator that checks threshold, anomaly, and SLO burn rate conditions, and dispatches notifications to configured channels.

**Design:**

```go
// internal/alerting/evaluator.go
package alerting

type AlertEvaluator struct {
    ruleStore     *postgres.AlertRuleStore
    metricService *query.REDMetricService
    sloStore      *postgres.SLOStore
    notifier      *Notifier
    incidentStore *postgres.IncidentStore
    interval      time.Duration // 30 seconds
}

type AlertCondition struct {
    Metric    string  `json:"metric"`    // "error_rate", "p99_latency", "request_rate"
    Operator  string  `json:"operator"`  // ">", "<", ">=", "<=", "=="
    Value     float64 `json:"value"`
    For       string  `json:"for"`       // "5m", "10m" — duration the condition must hold
}

type SLOBurnRateCondition struct {
    BurnRateWindow string  `json:"burn_rate_window"` // "1h"
    Threshold      float64 `json:"threshold"`        // 14.4 (Google SRE multi-window)
    LongWindow     string  `json:"long_window"`      // "6h"
    LongThreshold  float64 `json:"long_threshold"`   // 6.0
}

// EvaluateRule checks a single alert rule against current data.
// Returns true if the alert should fire.
func (e *AlertEvaluator) EvaluateRule(ctx context.Context, rule *model.AlertRule) (bool, error) {
    switch rule.RuleType {
    case "threshold":
        return e.evaluateThreshold(ctx, rule)
    case "slo_burn_rate":
        return e.evaluateBurnRate(ctx, rule)
    case "predictive":
        return false, nil // Phase 6
    default:
        return false, fmt.Errorf("unknown rule type: %s", rule.RuleType)
    }
}
```

```go
// internal/alerting/notifier.go
package alerting

type Notifier struct {
    channels map[string]NotificationChannel
}

type NotificationChannel interface {
    Send(ctx context.Context, alert *AlertNotification) error
    Type() string
}

type AlertNotification struct {
    RuleName    string
    Severity    string
    ServiceName string
    Summary     string // human-readable alert description
    Details     map[string]string
    FiredAt     time.Time
    DashboardURL string
}

// SlackChannel sends alerts to Slack via incoming webhook.
type SlackChannel struct {
    WebhookURL string
    Channel    string
}

func (s *SlackChannel) Send(ctx context.Context, alert *AlertNotification) error {
    // POST to webhook URL with Slack Block Kit payload
    // Include: severity emoji, rule name, service, summary, dashboard link
    return nil
}

// PagerDutyChannel creates PagerDuty incidents via Events API v2.
type PagerDutyChannel struct {
    RoutingKey string
}

func (p *PagerDutyChannel) Send(ctx context.Context, alert *AlertNotification) error {
    // POST to https://events.pagerduty.com/v2/enqueue
    // event_action: "trigger", severity mapping, dedup_key from rule ID
    return nil
}

// WebhookChannel sends alerts to arbitrary HTTP endpoints.
type WebhookChannel struct {
    URL     string
    Headers map[string]string
}

func (w *WebhookChannel) Send(ctx context.Context, alert *AlertNotification) error {
    // POST JSON payload to configured URL with custom headers
    return nil
}
```

```sql
-- internal/storage/postgres/migrations/004_alerts_and_incidents.sql

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
                    CHECK (rule_type IN ('threshold', 'anomaly', 'slo_burn_rate', 'predictive')),
    service_name    TEXT,
    slo_id          UUID REFERENCES slos(id),
    condition       JSONB NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('critical', 'warning', 'info')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_fired_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_rule_channels (
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES notification_channels(id) ON DELETE CASCADE,
    PRIMARY KEY (alert_rule_id, channel_id)
);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    title           TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('critical', 'warning', 'info')),
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
    event_type      TEXT NOT NULL
                    CHECK (event_type IN ('created', 'acknowledged', 'escalated',
                           'comment', 'ai_analysis', 'resolved', 'reopened')),
    user_id         UUID REFERENCES users(id),
    message         TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incidents_tenant_status ON incidents(tenant_id, status, started_at);
CREATE INDEX idx_incident_events_incident ON incident_events(incident_id);
```

**Testing:**

- Unit: Threshold evaluator fires when error_rate > 0.05 for 5 minutes
- Unit: Threshold evaluator does NOT fire when condition held for only 3 of 5 required minutes
- Unit: SLO burn rate evaluator fires when 1h burn_rate > 14.4 AND 6h burn_rate > 6.0 (multi-window)
- Integration: Create Slack channel, trigger alert, verify HTTP POST sent to webhook URL with correct payload
- Integration: Create PagerDuty channel, trigger alert, verify event sent to PagerDuty Events API
- Integration: Trigger alert, verify incident created in `incidents` table with correct severity and affected services
- Integration: Trigger same alert twice within 5 minutes (dedup window), verify only 1 incident created
- Integration: Resolve alert condition, verify incident status updated to "resolved"

---

## Phase 5: Log Ingestion, Correlation, and Unified Query

**Purpose:** Add log ingestion via OTLP, trace-to-log correlation using W3C Trace Context headers, and a unified search API that queries across spans, metrics, and logs in a single request.

### Task 5.1: Log Ingestion Pipeline

**What:** Extend the OTLP receiver to accept log records, write them to ClickHouse via Kafka, and support trace_id/span_id correlation.

**Design:**

```go
// internal/ingest/log_consumer.go
package ingest

type LogConsumer struct {
    reader    *kafka.Reader
    logWriter *clickhouse.LogWriter
    config    ConsumerConfig
}

// ClickHouse log table schema
// (created in ClickHouse migrations alongside spans)
```

```sql
-- ClickHouse log records table
CREATE TABLE log_records
(
    `tenant_id`             LowCardinality(String) CODEC(ZSTD(1)),
    `timestamp`             DateTime64(9) CODEC(DoubleDelta, LZ4),
    `observed_timestamp`    DateTime64(9) CODEC(DoubleDelta, LZ4),
    `trace_id`              FixedString(32) CODEC(ZSTD(1)),
    `span_id`               String CODEC(ZSTD(1)),
    `trace_flags`           UInt32 DEFAULT 0,
    `severity_text`         LowCardinality(String) CODEC(ZSTD(1)),
    `severity_number`       UInt8 CODEC(ZSTD(1)),
    `body`                  String CODEC(ZSTD(2)),
    `attributes_string`     Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    `attributes_number`     Map(LowCardinality(String), Float64) CODEC(ZSTD(1)),
    `resource_fingerprint`  String CODEC(ZSTD(1)),
    `resources_string`      Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    `scope_name`            LowCardinality(String) CODEC(ZSTD(1)),
    `scope_version`         LowCardinality(String) CODEC(ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, severity_number, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(30)
SETTINGS index_granularity = 8192;

ALTER TABLE log_records ADD INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 1;
ALTER TABLE log_records ADD INDEX idx_body body TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 1;
```

```go
// internal/query/log_query.go
package query

type LogQueryService struct {
    conn clickhouse.Conn
}

type LogSearchRequest struct {
    TenantID      string            `json:"tenant_id"`
    ServiceName   string            `json:"service_name,omitempty"`
    TraceID       string            `json:"trace_id,omitempty"`      // trace correlation
    SpanID        string            `json:"span_id,omitempty"`       // span correlation
    MinSeverity   int               `json:"min_severity,omitempty"`  // OTLP severity 1-24
    BodyContains  string            `json:"body_contains,omitempty"` // full-text search
    Attributes    map[string]string `json:"attributes,omitempty"`
    StartTime     time.Time         `json:"start_time"`
    EndTime       time.Time         `json:"end_time"`
    Limit         int               `json:"limit"`
}

// GetLogsForTrace returns all log records correlated to a specific trace.
func (s *LogQueryService) GetLogsForTrace(ctx context.Context, tenantID, traceID string) ([]model.LogRecord, error) { ... }
```

**Testing:**

- Unit: Log consumer correctly maps OTLP LogRecord severity_number (1-24) to severity_text
- Integration: Send logs via OTLP with trace_id set, query by trace_id, verify log-trace correlation
- Integration: Send 1,000 log records, search by `body_contains="connection refused"`, verify matching records
- Integration: Filter by `min_severity=17` (ERROR), verify only ERROR and FATAL logs returned
- Integration: Send logs without trace_id, verify they are stored and queryable by service/time

---

### Task 5.2: Unified Signal Correlation API

**What:** Build an API endpoint that returns correlated spans, logs, and metrics for a given trace or time window, enabling the "single pane of glass" experience.

**Design:**

```protobuf
// proto/api/v1/correlation_service.proto
service CorrelationService {
  // Returns spans, logs, and metrics correlated to a trace
  rpc GetCorrelatedSignals(GetCorrelatedSignalsRequest) returns (GetCorrelatedSignalsResponse);
}

message GetCorrelatedSignalsRequest {
  string trace_id = 1;
  // OR: time range + service for broader correlation
  string service_name = 2;
  int64 start_time_unix_ms = 3;
  int64 end_time_unix_ms = 4;
}

message GetCorrelatedSignalsResponse {
  repeated Span spans = 1;
  repeated LogRecord logs = 2;
  repeated REDMetricPoint metrics = 3;
  // Timeline: all signals sorted by timestamp for chronological view
  repeated TimelineEvent timeline = 4;
}

message TimelineEvent {
  string type = 1;       // "span", "log", "metric_anomaly", "deployment"
  int64 timestamp_ms = 2;
  string summary = 3;
  string source = 4;     // service name
  map<string, string> metadata = 5;
}
```

**Testing:**

- Integration: Insert trace with 5 spans, correlated logs (same trace_id), verify `GetCorrelatedSignals` returns both
- Integration: Verify timeline events are sorted chronologically across signal types
- Integration: Request correlation for a service over 1-hour window, verify RED metrics included
- Integration: Request with non-existent trace_id, verify empty response (not error)

---

## Phase 6: AI-Powered Root Cause Analysis

**Purpose:** Integrate LLM-based root cause analysis that generates plain-English incident narratives by correlating trace anomalies, metric spikes, log errors, and deployment events. This is the primary differentiating feature.

### Task 6.1: AI Analysis Pipeline

**What:** Build the LLM integration layer that collects correlated signals for an incident, constructs a structured prompt, calls the configured LLM, and stores the analysis result.

**Design:**

```go
// internal/ai/root_cause.go
package ai

type RootCauseAnalyzer struct {
    llmClient     *LLMClient
    correlator    *query.CorrelationService
    analysisStore *postgres.AIAnalysisStore
}

type AnalysisInput struct {
    IncidentID      uuid.UUID
    TenantID        uuid.UUID
    AffectedServices []string
    TimeWindow      TimeWindow
    Spans           []model.Span      // error/slow spans
    Logs            []model.LogRecord  // error logs in window
    Metrics         []REDMetricPoint   // RED metrics showing anomaly
    Topology        []ServiceDependency // relevant dependency edges
    RecentDeployments []Deployment      // deployments in the time window
}

type AnalysisResult struct {
    ID            uuid.UUID `db:"id"`
    IncidentID    uuid.UUID `db:"incident_id"`
    TenantID      uuid.UUID `db:"tenant_id"`
    AnalysisType  string    `db:"analysis_type"` // "root_cause"
    InputSignals  JSONB     `db:"input_signals"`
    Narrative     string    `db:"output_narrative"`
    ModelID       string    `db:"model_id"`
    Confidence    float64   `db:"confidence"`
    FeedbackRating *int     `db:"feedback_rating"` // user feedback 1-5
    TokensUsed    int       `db:"tokens_used"`
    LatencyMs     int       `db:"latency_ms"`
    CreatedAt     time.Time `db:"created_at"`
}

// Analyze generates a root cause narrative for an incident.
func (a *RootCauseAnalyzer) Analyze(ctx context.Context, input *AnalysisInput) (*AnalysisResult, error) {
    // 1. Collect correlated signals (spans, logs, metrics)
    // 2. Build structured prompt with signal data
    // 3. Call LLM via LLMClient
    // 4. Parse response and extract narrative + confidence
    // 5. Store analysis result in PostgreSQL
    // 6. Update incident.ai_summary
    return nil, nil
}
```

```go
// internal/ai/llm_client.go
package ai

type LLMClient struct {
    baseURL    string // LiteLLM proxy URL or direct API
    apiKey     string
    modelID    string // e.g., "claude-opus-4-6", "gpt-4o"
    httpClient *http.Client
    timeout    time.Duration // 60s
}

type LLMRequest struct {
    Model    string          `json:"model"`
    Messages []LLMMessage    `json:"messages"`
    MaxTokens int            `json:"max_tokens"`
    Temperature float64      `json:"temperature"` // 0.1 for deterministic analysis
}

type LLMMessage struct {
    Role    string `json:"role"`    // "system", "user"
    Content string `json:"content"`
}

// BuildRootCausePrompt constructs the system + user prompt.
func BuildRootCausePrompt(input *AnalysisInput) []LLMMessage {
    systemPrompt := `You are an expert SRE analyzing a service mesh incident.
Given the following observability signals, provide:
1. A plain-English summary of what happened (2-3 sentences)
2. The most likely root cause with supporting evidence
3. A timeline of events leading to the incident
4. Recommended remediation steps

Format your response as JSON:
{
  "summary": "...",
  "root_cause": "...",
  "timeline": [{"time": "...", "event": "..."}],
  "confidence": 0.0-1.0,
  "remediation": ["..."]
}`

    userPrompt := fmt.Sprintf(`Incident: %s
Affected services: %s
Time window: %s to %s

Error spans (%d):
%s

Error logs (%d):
%s

Metric anomalies:
%s

Service topology:
%s

Recent deployments:
%s`,
        input.IncidentID,
        strings.Join(input.AffectedServices, ", "),
        input.TimeWindow.Start.Format(time.RFC3339),
        input.TimeWindow.End.Format(time.RFC3339),
        len(input.Spans), formatSpans(input.Spans),
        len(input.Logs), formatLogs(input.Logs),
        formatMetrics(input.Metrics),
        formatTopology(input.Topology),
        formatDeployments(input.RecentDeployments),
    )

    return []LLMMessage{
        {Role: "system", Content: systemPrompt},
        {Role: "user", Content: userPrompt},
    }
}
```

```sql
-- internal/storage/postgres/migrations/005_ai_analyses.sql

CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    incident_id     UUID REFERENCES incidents(id),
    analysis_type   TEXT NOT NULL
                    CHECK (analysis_type IN ('root_cause', 'prediction', 'governance_audit')),
    input_signals   JSONB NOT NULL,
    output_narrative TEXT NOT NULL,
    model_id        TEXT NOT NULL,
    confidence      NUMERIC(4,3),
    feedback_rating SMALLINT,
    tokens_used     INT,
    latency_ms      INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_analyses_incident ON ai_analyses(incident_id);
CREATE INDEX idx_ai_analyses_tenant ON ai_analyses(tenant_id, created_at);
```

**Testing:**

- Unit: `BuildRootCausePrompt` includes all affected services, error spans, and deployment info in the prompt
- Unit: `BuildRootCausePrompt` truncates span/log data to fit within max token budget (e.g., 100K context)
- Unit: LLM response JSON parsing extracts narrative, confidence, timeline correctly
- Integration: Mock LLM endpoint, trigger analysis for an incident with known signals, verify stored result
- Integration: Verify incident.ai_summary is updated after analysis completes
- Integration: Submit feedback rating (1-5) on analysis, verify feedback_rating updated in database
- Integration: LLM timeout (>60s) returns graceful error without crashing the analysis pipeline

---

### Task 6.2: Automatic Analysis Triggering

**What:** Wire the AI analyzer into the incident lifecycle so that analysis is automatically triggered when a new critical incident is created.

**Design:**

```go
// internal/alerting/evaluator.go (addition)

// After creating an incident from a fired alert:
func (e *AlertEvaluator) onIncidentCreated(ctx context.Context, incident *model.Incident) {
    if incident.Severity == "critical" {
        // Enqueue async AI analysis
        go func() {
            input := e.collectAnalysisInput(ctx, incident)
            result, err := e.aiAnalyzer.Analyze(ctx, input)
            if err != nil {
                log.Error("AI analysis failed", "incident_id", incident.ID, "error", err)
                return
            }
            // Add incident event
            e.incidentStore.AddEvent(ctx, incident.ID, model.IncidentEvent{
                EventType: "ai_analysis",
                Message:   result.Narrative,
                Metadata:  map[string]interface{}{"confidence": result.Confidence, "model": result.ModelID},
            })
        }()
    }
}
```

**Testing:**

- Integration: Create critical incident, verify AI analysis is triggered automatically within 5 seconds
- Integration: Create warning incident, verify AI analysis is NOT triggered automatically
- Integration: Manually request analysis for a warning incident via API, verify it runs
- Integration: AI analysis failure does not prevent incident from being created/displayed

---

## Phase 7: Frontend — Topology, Traces, and Dashboards

**Purpose:** Build the React frontend with service topology visualization, trace waterfall view, RED metric dashboards, SLO error budget display, incident management, and AI analysis display.

### Task 7.1: Service Topology Visualization

**What:** Build an interactive force-directed graph showing service dependencies with health-colored nodes and latency/error-rate annotations on edges.

**Design:**

```typescript
// web/src/components/topology/TopologyGraph.tsx
import React from 'react';
import ForceGraph2D from 'react-force-graph-2d';

interface TopologyNode {
  id: string;           // service_name
  namespace: string;
  requestRate: number;
  errorRate: number;
  p99LatencyMs: number;
  healthStatus: 'healthy' | 'degraded' | 'critical';
}

interface TopologyEdge {
  source: string;
  target: string;
  protocol: string;
  callCount: number;
  errorRate: number;
  p99LatencyMs: number;
}

interface TopologyGraphProps {
  nodes: TopologyNode[];
  edges: TopologyEdge[];
  onNodeClick: (node: TopologyNode) => void;
  selectedNode?: string;
}

const nodeColor = (status: string): string => {
  switch (status) {
    case 'critical': return '#ef4444'; // red
    case 'degraded': return '#f59e0b'; // amber
    default: return '#22c55e';          // green
  }
};

const TopologyGraph: React.FC<TopologyGraphProps> = ({ nodes, edges, onNodeClick }) => {
  return (
    <ForceGraph2D
      graphData={{ nodes, links: edges }}
      nodeLabel={(node) => `${node.id}\n${node.requestRate.toFixed(1)} req/s`}
      nodeColor={(node) => nodeColor(node.healthStatus)}
      linkLabel={(link) => `${link.protocol} | p99: ${link.p99LatencyMs}ms`}
      linkColor={(link) => link.errorRate > 0.01 ? '#ef4444' : '#6b7280'}
      linkWidth={(link) => Math.max(1, Math.log10(link.callCount))}
      onNodeClick={onNodeClick}
    />
  );
};
```

**Testing:**

- Unit: `nodeColor` returns red for critical, amber for degraded, green for healthy
- e2e (Playwright): Load topology page with 5-service mock data, verify graph renders with 5 nodes
- e2e: Click a node, verify detail panel opens showing service name, request rate, error rate
- e2e: Verify edge between services shows protocol label on hover
- e2e: Verify critical node is visually distinct (red color)

---

### Task 7.2: Trace Waterfall View

**What:** Build the trace waterfall/timeline visualization showing spans as horizontal bars with parent-child nesting, duration labels, and error highlighting.

**Design:**

```typescript
// web/src/components/traces/TraceWaterfall.tsx

interface WaterfallSpan {
  spanId: string;
  parentSpanId: string | null;
  serviceName: string;
  operationName: string;
  startTimeMs: number;      // relative to trace start
  durationMs: number;
  statusCode: number;
  hasError: boolean;
  depth: number;            // nesting depth (computed)
  attributes: Record<string, string>;
}

interface TraceWaterfallProps {
  traceId: string;
  spans: WaterfallSpan[];
  totalDurationMs: number;
  onSpanClick: (span: WaterfallSpan) => void;
}

// Span bar rendering:
// - x position = (startTimeMs / totalDurationMs) * containerWidth
// - width = (durationMs / totalDurationMs) * containerWidth
// - y position = depth * rowHeight
// - color = service color (consistent hash-based palette)
// - border = red for error spans
// - label = operationName + durationMs
```

**Testing:**

- e2e: Load trace view with 10-span mock trace, verify waterfall renders 10 bars
- e2e: Verify root span starts at x=0 and spans the full width
- e2e: Verify child spans are indented below their parent
- e2e: Click a span bar, verify detail panel shows attributes, service name, duration
- e2e: Verify error spans have red border/indicator

---

### Task 7.3: RED Metric Dashboards and SLO Display

**What:** Build dashboard pages showing RED metric time-series charts per service and SLO error budget gauges.

**Design:**

```typescript
// web/src/components/metrics/REDDashboard.tsx
import * as echarts from 'echarts';

interface REDDashboardProps {
  serviceName: string;
  dataPoints: REDMetricPoint[];
  timeRange: { start: Date; end: Date };
}

// Three stacked charts:
// 1. Request Rate (line chart, requests/sec)
// 2. Error Rate (area chart, percentage, red fill when > threshold)
// 3. Latency Percentiles (multi-line: p50, p95, p99)

// web/src/components/slos/SLODashboard.tsx
interface SLOGaugeProps {
  sloName: string;
  targetPercent: number;    // e.g., 99.95
  currentSLI: number;       // e.g., 99.87
  errorBudgetRemaining: number; // e.g., 0.03 (3%)
  burnRate1h: number;
  burnRate6h: number;
}

// Error budget gauge: circular progress indicator
// - Green when > 50% budget remaining
// - Amber when 10-50% remaining
// - Red when < 10% remaining
// Burn rate sparkline: 24h rolling burn rate chart
```

**Testing:**

- e2e: Load RED dashboard for a service with 1-hour of data, verify 3 charts render
- e2e: Verify error rate chart shows red fill when error rate exceeds 5%
- e2e: Load SLO dashboard, verify gauge shows correct SLI value and color
- e2e: Verify burn rate values displayed match API response
- e2e: Change time range picker, verify charts re-render with new data

---

### Task 7.4: Incident Management UI with AI Summary

**What:** Build the incident list, detail view, and AI root cause analysis display.

**Design:**

```typescript
// web/src/components/incidents/IncidentDetail.tsx

interface IncidentDetailProps {
  incident: {
    id: string;
    title: string;
    severity: 'critical' | 'warning' | 'info';
    status: 'open' | 'acknowledged' | 'investigating' | 'resolved';
    affectedServices: string[];
    aiSummary?: string;
    aiConfidence?: number;
    aiModel?: string;
    timeline: IncidentEvent[];
    startedAt: Date;
    resolvedAt?: Date;
  };
  onAcknowledge: () => void;
  onResolve: () => void;
  onRequestAnalysis: () => void;
  onSubmitFeedback: (rating: number) => void;
}

// Layout:
// - Header: title, severity badge, status badge, action buttons
// - AI Summary panel: narrative text, confidence score, model info, feedback thumbs
// - Affected Services: clickable list linking to topology view
// - Timeline: chronological list of events (created, acknowledged, AI analysis, comments)
// - Correlated Signals: tabbed view of related spans, logs, metrics
```

**Testing:**

- e2e: Load incident detail with AI summary, verify narrative text is displayed
- e2e: Click "Acknowledge" button, verify status changes to "acknowledged"
- e2e: Click "Request Analysis" button, verify loading state, then AI summary appears
- e2e: Submit feedback rating (thumbs up/down), verify feedback is persisted
- e2e: Verify timeline shows events in chronological order

---

## Phase 8: Intelligent Adaptive Sampling

**Purpose:** Build the AI-guided trace sampling system that learns which trace patterns are diagnostically novel and prioritizes their retention, reducing storage costs 70-90% without sacrificing debugging fidelity.

### Task 8.1: Sampling Rule Engine

**What:** Build the rule-based sampling engine that evaluates trace properties against configured rules (always_sample, rate_limit, never_sample) before the AI sampler.

**Design:**

```go
// internal/sampling/rule_engine.go
package sampling

type SamplingDecision string
const (
    DecisionKeep   SamplingDecision = "keep"
    DecisionDrop   SamplingDecision = "drop"
    DecisionDefer  SamplingDecision = "defer" // let next evaluator decide
)

type SamplingRule struct {
    ID        uuid.UUID `db:"id"`
    TenantID  uuid.UUID `db:"tenant_id"`
    ServiceName *string `db:"service_name"` // nil = global
    RuleType  string    `db:"rule_type"`
    Condition JSONB     `db:"condition"`
    Priority  int       `db:"priority"` // higher = evaluated first
    IsActive  bool      `db:"is_active"`
}

type RuleEngine struct {
    rules []SamplingRule // sorted by priority descending
}

// Evaluate checks a span against all rules in priority order.
// First matching rule wins. If no rule matches, returns Defer.
func (e *RuleEngine) Evaluate(span *model.Span) (SamplingDecision, *SamplingRule) {
    for _, rule := range e.rules {
        if rule.Matches(span) {
            switch rule.RuleType {
            case "always_sample":
                return DecisionKeep, &rule
            case "never_sample":
                return DecisionDrop, &rule
            case "rate_limit":
                // Check rate limiter for this service
                return e.checkRateLimit(span, &rule)
            case "ai_adaptive":
                return DecisionDefer, &rule // handled by AI sampler
            }
        }
    }
    return DecisionDefer, nil
}
```

```sql
-- internal/storage/postgres/migrations/006_sampling.sql

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sampling_decisions (
    trace_id        FixedString(32),
    tenant_id       String,
    rule_id         Nullable(UUID),
    decision        LowCardinality(String),  -- 'keep', 'drop'
    novelty_score   Nullable(Float32),
    reason          Nullable(String),
    decided_at      DateTime64(9)
)
ENGINE = MergeTree
PARTITION BY toDate(decided_at)
ORDER BY (tenant_id, decided_at)
TTL toDateTime(decided_at) + toIntervalDay(7)
SETTINGS index_granularity = 8192;
```

**Testing:**

- Unit: `always_sample` rule returns Keep for matching spans
- Unit: Rules evaluated in priority order; highest priority rule wins
- Unit: `rate_limit` rule with `max_traces_per_second=100` drops excess traces
- Unit: No matching rule returns Defer
- Integration: Configure rules, send diverse spans, verify sampling decisions stored in ClickHouse

---

### Task 8.2: AI Novelty Scorer

**What:** Build an AI-based novelty scorer that computes a 0-1 score for each trace based on how diagnostically novel or unusual it is compared to recently observed patterns.

**Design:**

```go
// internal/sampling/ai_sampler.go
package sampling

type AISampler struct {
    llmClient   *ai.LLMClient
    spanReader  *clickhouse.SpanReader
    config      AISamplerConfig
}

type AISamplerConfig struct {
    NoveltyThreshold float32 // 0.7 — keep traces with novelty >= this
    ErrorBias        float32 // 2.0 — multiply novelty score for error traces
    LatencyPctBias   float32 // 0.95 — bias toward high-latency traces
    BatchSize        int     // 100 — evaluate traces in batches
    ModelID          string  // LLM model for novelty scoring
}

// ScoreBatch evaluates a batch of trace summaries for novelty.
// Uses a lightweight LLM prompt that classifies traces into:
// - routine (0.0-0.3): common happy-path patterns
// - interesting (0.3-0.7): unusual but not diagnostic
// - novel (0.7-1.0): rare errors, extreme latencies, unusual call patterns
func (s *AISampler) ScoreBatch(ctx context.Context, traces []TraceSummary) ([]float32, error) {
    // 1. Build batch prompt with trace summaries
    // 2. Call LLM with structured output format
    // 3. Parse novelty scores
    // 4. Apply error bias: if trace has_error, score *= ErrorBias (capped at 1.0)
    // 5. Apply latency bias: if trace duration > P95, score *= 1.5 (capped at 1.0)
    return nil, nil
}

type TraceSummary struct {
    TraceID      string
    ServiceCount int
    SpanCount    int
    DurationMs   float64
    HasError     bool
    ErrorTypes   []string // unique exception types
    Services     []string // service call chain
    HTTPStatuses []int    // unique HTTP status codes
}
```

**Testing:**

- Unit: Error traces get novelty score multiplied by ErrorBias (capped at 1.0)
- Unit: Traces with duration > P95 get latency bias applied
- Unit: Routine traces (200 OK, normal latency, common services) score < 0.3
- Integration: Score batch of 100 traces, verify all return valid scores 0.0-1.0
- Integration: Sampling decisions (keep/drop + novelty_score) stored in ClickHouse `sampling_decisions`
- Integration: Monitor sampling rate over 10,000 traces, verify 70-90% reduction from baseline

---

## Phase 9: Predictive SLO Breach Detection

**Purpose:** Build the predictive model that forecasts SLO breaches 5-30 minutes before they occur by analyzing early warning signals (memory growth, queue depths, upstream latency trends).

### Task 9.1: Time-Series Feature Extraction

**What:** Build a feature extractor that computes rolling statistics from RED metrics and system metrics to feed the prediction model.

**Design:**

```go
// internal/slo/predictor.go
package slo

type SLOPredictor struct {
    metricReader *clickhouse.MetricReader
    sloStore     *postgres.SLOStore
    llmClient    *ai.LLMClient
    config       PredictorConfig
}

type PredictorConfig struct {
    LeadTimeMinutes  int     // 15 — predict this far ahead
    EvaluationWindow int     // 60 — minutes of historical data to analyze
    ConfidenceThreshold float64 // 0.8 — minimum confidence to trigger alert
}

type PredictionFeatures struct {
    SLOID               uuid.UUID
    ServiceName         string
    CurrentSLI          float64
    ErrorBudgetRemaining float64
    BurnRate1h          float64
    BurnRate6h          float64
    // Trend features (computed from rolling windows)
    ErrorRateTrend5m    float64 // slope of error rate over last 5 minutes
    LatencyTrend5m      float64 // slope of p99 latency over last 5 minutes
    RequestRateTrend5m  float64 // slope of request rate (traffic change)
    ErrorRateVariance   float64 // variance of error rate (stability)
    // Upstream signals
    UpstreamErrorRates  map[string]float64 // error rates of upstream dependencies
    UpstreamLatencyP99  map[string]float64 // p99 latency of upstream dependencies
}

// Predict evaluates whether an SLO is likely to breach within LeadTimeMinutes.
func (p *SLOPredictor) Predict(ctx context.Context, slo *model.SLO) (*PredictionResult, error) {
    features := p.extractFeatures(ctx, slo)
    // Use LLM to analyze trends and predict breach likelihood
    // Prompt includes: current SLI, burn rates, trend slopes, upstream health
    return p.evaluateWithLLM(ctx, features)
}

type PredictionResult struct {
    SLOID           uuid.UUID
    WillBreach      bool
    BreachTimeEst   time.Time // estimated breach time
    Confidence      float64
    ContributingFactors []string // e.g., ["upstream payment-db latency increasing", "error rate accelerating"]
    Narrative       string
}
```

**Testing:**

- Unit: Feature extraction computes correct 5-minute error rate slope from 5 data points
- Unit: Upstream error rates are collected from dependency graph
- Integration: Feed increasing error rate trend (1%, 2%, 3%, 4%, 5% over 5 minutes), verify prediction fires
- Integration: Feed stable error rate (0.1% constant), verify no prediction fires
- Integration: Prediction with `confidence < 0.8` does not trigger alert
- Integration: Prediction result stored as ai_analysis with `analysis_type='prediction'`

---

### Task 9.2: Predictive Alert Integration

**What:** Wire predictive SLO breach detection into the alert rule engine so that `rule_type='predictive'` alerts fire when the predictor forecasts a breach.

**Design:**

```go
// internal/alerting/evaluator.go (addition)

func (e *AlertEvaluator) evaluatePredictive(ctx context.Context, rule *model.AlertRule) (bool, error) {
    slo, err := e.sloStore.Get(ctx, rule.SLOID)
    if err != nil { return false, err }

    prediction, err := e.sloPredictor.Predict(ctx, slo)
    if err != nil { return false, err }

    if prediction.WillBreach && prediction.Confidence >= e.config.PredictiveConfidenceThreshold {
        // Create incident with prediction details
        e.createPredictiveIncident(ctx, rule, slo, prediction)
        return true, nil
    }
    return false, nil
}
```

**Testing:**

- Integration: Create predictive alert rule, simulate deteriorating metrics, verify alert fires
- Integration: Verify incident created includes AI narrative explaining the predicted breach
- Integration: Verify incident title includes "Predicted" to distinguish from actual breaches
- Integration: Prediction that does not materialize (false positive) is logged for model feedback

---

## Phase 10: eBPF Network Flow Agent

**Purpose:** Build the eBPF-based per-node agent that captures L4/L7 network flows without sidecars, enabling topology inference for uninstrumented services.

### Task 10.1: eBPF Flow Capture Agent

**What:** Build a Go agent that loads eBPF programs to capture TCP/UDP network flows at the kernel level, parsing L7 protocols (HTTP, gRPC, DNS).

**Design:**

```go
// internal/ebpf/loader.go
package ebpf

import (
    "github.com/cilium/ebpf"
    "github.com/cilium/ebpf/link"
)

type FlowAgent struct {
    objs      *ebpfObjects  // generated by bpf2go
    tcLinks   []link.Link
    flowChan  chan *NetworkFlow
    config    AgentConfig
}

type AgentConfig struct {
    Interfaces   []string      // e.g., ["eth0", "veth+"]
    BPFDir       string        // path to compiled BPF objects
    FlowBufSize  int           // ring buffer size (default: 4096)
    BatchInterval time.Duration // 1 second
    ExportTarget  string       // OTLP endpoint or Kafka broker
}

type NetworkFlow struct {
    Timestamp    time.Time
    SourceIP     net.IP
    SourcePort   uint16
    DestIP       net.IP
    DestPort     uint16
    Protocol     string // "TCP", "UDP"
    L7Protocol   string // "HTTP", "gRPC", "DNS", "Kafka", ""
    HTTPMethod   string
    HTTPStatus   uint16
    HTTPPath     string
    DNSQuery     string
    BytesSent    uint64
    BytesRecv    uint64
    Duration     time.Duration
    Verdict      string // "allowed", "denied"
}

// Start loads eBPF programs, attaches to network interfaces,
// and begins reading flows from the kernel ring buffer.
func (a *FlowAgent) Start(ctx context.Context) error { ... }

// enrichFlow resolves pod/service names from IP addresses
// using Kubernetes API or local cache.
func (a *FlowAgent) enrichFlow(flow *NetworkFlow) *EnrichedFlow { ... }
```

```c
// internal/ebpf/programs/network_flow.c
// BPF program attached to TC (traffic control) ingress/egress hooks.
// Captures per-packet metadata and sends to userspace via ring buffer.

#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>

struct flow_event {
    __u64 timestamp;
    __u32 src_ip;
    __u32 dst_ip;
    __u16 src_port;
    __u16 dst_port;
    __u8  protocol;
    __u32 bytes;
    // L7 fields populated by separate HTTP/DNS parsers
    __u16 http_status;
    __u8  http_method;  // 0=unknown, 1=GET, 2=POST, etc.
};

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24); // 16MB ring buffer
} flow_events SEC(".maps");

SEC("tc")
int capture_flow(struct __sk_buff *skb) {
    // Parse Ethernet -> IP -> TCP/UDP headers
    // Extract 5-tuple (src_ip, src_port, dst_ip, dst_port, proto)
    // Submit to ring buffer
    return TC_ACT_OK; // pass-through, don't modify packet
}
```

**Testing:**

- Unit: L7 parser correctly identifies HTTP GET/POST from TCP payload first bytes
- Unit: DNS parser extracts query name from DNS request packets
- Integration: Run eBPF agent in a test namespace, generate HTTP traffic, verify flows captured
- Integration: Verify Kubernetes pod-to-service name resolution from IP addresses
- Integration: Verify flows are exported to ClickHouse `network_flows` table
- Integration: Agent handles interface hot-plug (new veth interfaces for new pods)

---

### Task 10.2: eBPF-to-Topology Integration

**What:** Build the pipeline that converts eBPF network flows into service dependency edges, enriching the topology with uninstrumented service connections.

**Design:**

```go
// internal/topology/ebpf_enricher.go
package topology

type EBPFEnricher struct {
    flowReader *clickhouse.FlowReader
    depStore   *postgres.DependencyStore
    interval   time.Duration // 5 minutes
}

// Run periodically scans network_flows for new source->dest pairs
// not already in the trace-derived topology and creates edges
// with discovered_via='ebpf'.
func (e *EBPFEnricher) Run(ctx context.Context) error { ... }
```

```sql
-- ClickHouse network flows table
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
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)
ORDER BY (tenant_id, source_service, dest_service, timestamp)
TTL toDateTime(timestamp) + toIntervalDay(14)
SETTINGS index_granularity = 8192;
```

**Testing:**

- Integration: Generate network flows for pod A -> pod B (no OTLP spans), verify dependency edge created with `discovered_via='ebpf'`
- Integration: eBPF-discovered edge appears in topology API alongside trace-discovered edges
- Integration: Flow for already-known trace-derived dependency does NOT create duplicate edge
- Integration: Topology graph shows eBPF-discovered edges with distinct visual indicator

---

## Phase 11: Mesh Governance and Configuration Auditing

**Purpose:** Build the AI-powered mesh governance feature that audits Istio/Linkerd/Cilium configurations across teams, detects anti-patterns (excessive retries, missing timeouts, inconsistent circuit breakers), and proposes standardized baselines.

### Task 11.1: Mesh Configuration Collector

**What:** Build a Kubernetes controller that watches Istio VirtualService, DestinationRule, AuthorizationPolicy, and Cilium NetworkPolicy resources and stores their configurations for audit.

**Design:**

```go
// internal/governance/collector.go
package governance

type MeshConfigCollector struct {
    k8sClient     kubernetes.Interface
    istioClient   istioclient.Interface
    configStore   *postgres.MeshConfigStore
}

type MeshConfig struct {
    ID            uuid.UUID `db:"id"`
    TenantID      uuid.UUID `db:"tenant_id"`
    ResourceType  string    `db:"resource_type"` // "VirtualService", "DestinationRule", etc.
    Name          string    `db:"name"`
    Namespace     string    `db:"namespace"`
    ServiceName   string    `db:"service_name"`
    Spec          JSONB     `db:"spec"`
    // Extracted policy fields for indexing
    RetryAttempts *int      `db:"retry_attempts"`
    TimeoutMs     *int      `db:"timeout_ms"`
    CircuitBreaker *bool    `db:"has_circuit_breaker"`
    MTLS          *string   `db:"mtls_mode"` // "STRICT", "PERMISSIVE", "DISABLE"
    CollectedAt   time.Time `db:"collected_at"`
}
```

```sql
-- internal/storage/postgres/migrations/007_mesh_governance.sql

CREATE TABLE mesh_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    resource_type   TEXT NOT NULL,
    name            TEXT NOT NULL,
    namespace       TEXT NOT NULL,
    service_name    TEXT,
    spec            JSONB NOT NULL,
    retry_attempts  INT,
    timeout_ms      INT,
    has_circuit_breaker BOOLEAN,
    mtls_mode       TEXT,
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, name, namespace)
);

CREATE INDEX idx_mesh_configs_service ON mesh_configs(tenant_id, service_name);
```

**Testing:**

- Integration: Deploy Istio VirtualService in test cluster, verify config collected and stored
- Integration: Update VirtualService retries from 3 to 10, verify new config stored with updated retry_attempts
- Integration: Verify DestinationRule circuit breaker settings extracted into `has_circuit_breaker` field

---

### Task 11.2: Governance Audit and Recommendations

**What:** Build an AI-powered auditor that analyzes collected mesh configurations, detects anti-patterns, and generates recommendations with severity ratings.

**Design:**

```go
// internal/governance/auditor.go
package governance

type GovernanceAuditor struct {
    configStore *postgres.MeshConfigStore
    llmClient   *ai.LLMClient
    depStore    *postgres.DependencyStore
}

type AuditFinding struct {
    Severity    string // "critical", "warning", "info"
    Category    string // "retry_storm", "missing_timeout", "inconsistent_mtls", etc.
    Service     string
    Resource    string
    Description string
    Recommendation string
    Evidence    map[string]interface{}
}

// Audit runs a governance check across all mesh configs for a tenant.
// Rule-based checks (deterministic) + LLM analysis (contextual).
func (a *GovernanceAuditor) Audit(ctx context.Context, tenantID uuid.UUID) ([]AuditFinding, error) {
    configs, _ := a.configStore.ListByTenant(ctx, tenantID)
    deps, _ := a.depStore.ListByTenant(ctx, tenantID)

    var findings []AuditFinding

    // Rule-based checks
    findings = append(findings, a.checkExcessiveRetries(configs)...)
    findings = append(findings, a.checkMissingTimeouts(configs)...)
    findings = append(findings, a.checkInconsistentMTLS(configs)...)
    findings = append(findings, a.checkMissingCircuitBreakers(configs, deps)...)

    // LLM-based contextual analysis
    llmFindings, _ := a.llmAnalyze(ctx, configs, deps)
    findings = append(findings, llmFindings...)

    return findings, nil
}

// checkExcessiveRetries flags services with retryAttempts > 5
// (anti-pattern: retry storms can amplify failures).
func (a *GovernanceAuditor) checkExcessiveRetries(configs []MeshConfig) []AuditFinding { ... }
```

**Testing:**

- Unit: `checkExcessiveRetries` flags config with retryAttempts=10, severity="critical"
- Unit: `checkMissingTimeouts` flags VirtualService without timeout set
- Unit: `checkInconsistentMTLS` flags namespace with mixed STRICT/PERMISSIVE modes
- Integration: Audit tenant with known anti-patterns, verify all expected findings returned
- Integration: LLM analysis identifies contextual issues (e.g., retry policy on a database call)

---

## Phase 12: Production Hardening, Helm Charts, and Documentation

**Purpose:** Prepare the platform for production deployment with Helm charts, horizontal scaling, data retention automation, security hardening, and comprehensive documentation.

### Task 12.1: Helm Chart and Kubernetes Deployment

**What:** Create a production Helm chart with configurable replicas, resource limits, PersistentVolumeClaims, and Kubernetes-native health probes for all services.

**Design:**

```yaml
# deploy/helm/service-mesh-obs/values.yaml
global:
  tenantMode: "multi"  # "single" or "multi"
  storageClass: "gp3"

collector:
  replicas: 3
  resources:
    requests: { cpu: "500m", memory: "512Mi" }
    limits: { cpu: "2", memory: "2Gi" }
  otlp:
    grpcPort: 4317
    httpPort: 4318
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPU: 70

apiServer:
  replicas: 2
  resources:
    requests: { cpu: "250m", memory: "256Mi" }
    limits: { cpu: "1", memory: "1Gi" }

queryEngine:
  replicas: 2
  resources:
    requests: { cpu: "500m", memory: "512Mi" }
    limits: { cpu: "2", memory: "2Gi" }

aiAnalyzer:
  replicas: 1
  llm:
    provider: "litellm"  # or "anthropic", "openai"
    model: "claude-sonnet-4-20250514"
    apiKeySecretRef: "llm-api-key"
    maxConcurrent: 5
    timeoutSeconds: 60

sloEvaluator:
  replicas: 1

topologyBuilder:
  replicas: 1

alerter:
  replicas: 2

samplingController:
  replicas: 1

ebpfAgent:
  enabled: false  # DaemonSet, opt-in
  resources:
    requests: { cpu: "100m", memory: "128Mi" }
    limits: { cpu: "500m", memory: "256Mi" }

postgresql:
  # Using Bitnami PostgreSQL subchart
  auth:
    database: meshobs
    existingSecret: "meshobs-pg-secret"
  primary:
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
    persistence:
      size: 20Gi

clickhouse:
  # Using Altinity ClickHouse Operator or Bitnami subchart
  shards: 1
  replicas: 2
  resources:
    requests: { cpu: "2", memory: "4Gi" }
  storage:
    size: 100Gi

kafka:
  # Using Strimzi Kafka Operator or Bitnami subchart
  replicas: 3
  storage:
    size: 50Gi
```

**Testing:**

- Integration: `helm install` in a test cluster, verify all pods reach Ready state within 5 minutes
- Integration: Send OTLP data, query traces, verify end-to-end pipeline works in Kubernetes
- Integration: `helm upgrade` with new image tag, verify zero-downtime rolling update
- Integration: HPA scales collector from 2 to 4 replicas under load
- Integration: Delete a collector pod, verify it restarts and resumes processing (no data loss)

---

### Task 12.2: Data Retention and Partition Management

**What:** Build automated data retention that drops old ClickHouse partitions and PostgreSQL partitions based on tenant-configurable retention policies.

**Design:**

```go
// internal/storage/retention.go
package storage

type RetentionManager struct {
    chConn      clickhouse.Conn
    pgPool      *pgxpool.Pool
    tenantStore *postgres.TenantStore
    interval    time.Duration // 1 hour
}

// RunRetention drops ClickHouse partitions older than the tenant's retention_days.
// For multi-tenant, uses the shortest retention across all tenants as the minimum,
// then marks individual tenant data for deletion.
func (m *RetentionManager) RunRetention(ctx context.Context) error {
    // 1. Get minimum retention_days across all tenants
    // 2. Drop ClickHouse partitions older than min retention
    //    ALTER TABLE spans DROP PARTITION 'YYYY-MM-DD'
    // 3. For tenants with shorter retention, delete their specific data
    //    (rare case — most tenants use default 30 days)
    return nil
}
```

**Testing:**

- Integration: Set retention to 7 days, insert data 10 days ago, run retention, verify old data removed
- Integration: Verify active data (within retention window) is NOT removed
- Integration: Verify ClickHouse `system.parts` shows reduced partition count after retention

---

### Task 12.3: Security Hardening

**What:** Implement API key authentication with scoped permissions, rate limiting, and audit logging for all configuration changes.

**Design:**

```go
// internal/auth/middleware.go
package auth

type AuthMiddleware struct {
    apiKeyStore *postgres.APIKeyStore
    rateLimiter *RateLimiter
}

// Authenticate extracts the API key from the Authorization header,
// validates it, and sets tenant context for downstream handlers.
func (m *AuthMiddleware) Authenticate(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := extractAPIKey(r) // "Bearer <key>" or "X-API-Key: <key>"
        if key == "" {
            http.Error(w, "missing API key", http.StatusUnauthorized)
            return
        }

        apiKey, err := m.apiKeyStore.ValidateKey(r.Context(), key)
        if err != nil {
            http.Error(w, "invalid API key", http.StatusUnauthorized)
            return
        }

        // Rate limiting per tenant
        if !m.rateLimiter.Allow(apiKey.TenantID.String()) {
            http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
            return
        }

        // Set tenant context for PostgreSQL RLS
        ctx := context.WithValue(r.Context(), "tenant_id", apiKey.TenantID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

type RateLimiter struct {
    limiters sync.Map // map[tenantID]*rate.Limiter
    rps      int      // default: 1000 requests/sec per tenant
}
```

**Testing:**

- Unit: Valid API key passes authentication, sets correct tenant context
- Unit: Expired API key returns 401
- Unit: Rate limiter returns 429 when tenant exceeds configured RPS
- Integration: API key with `scopes=["read"]` cannot create SLOs (403 Forbidden)
- Integration: Audit log records all SLO/alert/tenant configuration changes

---

## Phase Summary & Dependency Graph

```
Phase 1: Foundation (Data Model, Ingestion, Storage)
   |
   v
Phase 2: Query Engine & Trace Reconstruction
   |
   +-----> Phase 3: Service Topology & Dependency Graph
   |          |
   v          v
Phase 4: SLO Engine, Alerting & Notifications
   |
   v
Phase 5: Log Ingestion, Correlation & Unified Query
   |
   +-----> Phase 6: AI-Powered Root Cause Analysis --------+
   |          |                                             |
   |          v                                             v
   |       Phase 7: Frontend (Topology, Traces, Dashboards)
   |          |
   v          v
Phase 8: Intelligent Adaptive Sampling
   |
   v
Phase 9: Predictive SLO Breach Detection
   |
   v
Phase 10: eBPF Network Flow Agent
   |
   v
Phase 11: Mesh Governance & Config Auditing
   |
   v
Phase 12: Production Hardening, Helm & Documentation
```

**Parallel execution opportunities:**
- Phase 3 (topology) can begin once Phase 1 ingestion is working, in parallel with Phase 2 query engine
- Phase 6 (AI analysis) and Phase 8 (sampling) can be developed in parallel once Phase 5 is complete
- Phase 7 (frontend) can begin skeleton work during Phase 3, with incremental feature additions through Phases 4-6
- Phase 10 (eBPF) is independent of Phases 6-9 and can be developed in parallel
- Phase 11 (governance) depends only on Phase 10 for topology data and Phase 6 for AI infrastructure

---

## Definition of Done

A phase is complete when:

1. **All task code is merged** to the main branch and passes CI (lint, unit tests, integration tests)
2. **Unit test coverage >= 80%** for all new packages (measured by `go test -cover`)
3. **Integration tests pass** against real PostgreSQL, ClickHouse, and Kafka instances (via testcontainers or Docker Compose test environment)
4. **API contracts are documented** — proto files are committed and generated clients are updated
5. **Database migrations are versioned** — all schema changes are in numbered migration files that run idempotently
6. **Docker Compose dev environment works** — `docker compose up` runs the full stack with the new phase's features
7. **Performance baselines are established** — latency p99 and throughput benchmarks recorded for critical paths:
   - Span ingestion: >= 50,000 spans/sec per collector instance
   - Trace query: < 200ms p99 for single trace retrieval
   - RED metric query: < 500ms p99 for 1-hour dashboard
   - Topology query: < 100ms p99
8. **Security requirements met** — all APIs require authentication, tenant isolation verified, no secrets in code
9. **Frontend components render correctly** (for UI phases) — Playwright e2e tests pass, responsive layout verified at 1280px and 1920px widths
10. **Runbook entry added** — operational documentation for monitoring, alerting on the new component, and rollback procedure
