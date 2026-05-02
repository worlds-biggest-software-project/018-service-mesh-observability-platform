# Service Mesh Observability Platform

Unified OpenTelemetry-native observability platform with AI-powered natural language root cause analysis, predictive SLO breach detection, and intelligent trace sampling — turning observability data into actionable narratives.

## The Problem

Current observability tools flood operators with data but not insights:

- **Operators see alerts; they don't understand causation** — "latency increased" but why? Which service? Which database?
- **Dashboards require expertise to interpret** — 50th vs. 95th percentile latency, tail latencies, upstream dependencies
- **Trace sampling is static and dumb** — either miss rare errors (head-based) or blow the budget (100% sampling)
- **SLO breaches are discovered reactively** — no tool predicts breaches 5–30 minutes ahead from early signals

Commercial platforms (Dynatrace at $58/host/month, Datadog at $31/host/month) solve this with AI/ML. Open-source tools (Jaeger, Tempo, SigNoz) provide the plumbing but no intelligence.

## What This Does

### Natural Language Root Cause Analysis (LLM-Native)
- **Generates plain-English incident summaries** — "Latency on the order-service → payment-gateway call increased 340ms starting at 14:23 UTC, correlated with a 12% drop in connection pool availability on payment-db, likely triggered by the batch job deployed at 14:18 UTC"
- **Correlates across signals** — trace spans, metrics, logs, deployments, in a single coherent narrative
- **Makes incident response accessible** — junior on-call engineers can understand and act without expertise
- **No open-source tool does this** — Dynatrace Davis AI is the only competitor

### Predictive SLO Breach Detection
- **Models trained on historical service behavior** predict breaches 5–30 minutes before they occur
- **Early warning signals** — memory growth rates, queue depths, upstream latency trends
- **Enables proactive mitigation** — rotate pods, shed load, page on-call *before* users are impacted
- **Reactive tools detect anomalies after they happen** — this is the gap

### Intelligent Adaptive Trace Sampling
- **Learns which trace patterns are diagnostically novel** — rare errors, extreme latencies, unusual sequences
- **Aggressively drops redundant happy-path traces** — reducing storage costs 70–90%
- **Preserves debugging fidelity** — never lose traces that matter for diagnosis
- **Current head-based and tail-based samplers are static** — no learning or adaptation

### Sidecar-Free Topology Inference (eBPF-Native)
- **Cilium Hubble approach** — kernel-level observability without pod modification
- **Captures network flows at L7** — understands HTTP method, status code, DNS queries, Kafka topics
- **Auto-infers service dependencies** for uninstrumented legacy services
- **LLM clustering** — groups traffic patterns to build topology without explicit instrumentation

## Key Differentiators

| Feature | This Platform | Jaeger | SigNoz | Grafana Tempo | Dynatrace | Datadog |
|---------|---|---|---|---|---|---|
| **LLM root cause narration** | ✓ | — | — | — | ✓ (Davis AI) | — |
| **Predictive SLO breach** | ✓ | — | — | — | — | — |
| **Intelligent sampling** | ✓ (AI-guided) | Adaptive only | — | — | — | — |
| **eBPF sidecar-free topo** | ✓ | — | — | — | — | — |
| **Unified logs/metrics/traces** | ✓ | — | ✓ | ✓ | ✓ | ✓ |
| **Open source** | ✓ | ✓ | ✓ (AGPLv3) | ✓ (AGPLv3) | — | — |
| **Cost** | Free | Free | $49+/mo | Free (50GB/mo) | $58/host/mo | $31/host/mo |

## Market & Opportunity

- **Market size**: Service mesh software $395M–$632M (2024–2026) → $3.6–$6.7B by 2033
- **Observability platform market**: $2.9B (2025) → $6.93B by 2031 at 15.62% CAGR
- **Buyers**: Platform/SRE engineers, backend managers, FinOps teams, regulated enterprises
- **Open-source gap**: No OSS tool provides LLM root cause analysis, predictive breach detection, or adaptive sampling

## Research Foundation

- **AI-driven anomaly detection in microservices** with knowledge graphs (ResearchGate 2024, MDPI 2024)
- **Dynamic resolution anomaly detection via LLM agents** — breaking the observability tax (ResearchGate 2024)
- **Performance benchmarks**: Istio +166% latency, Linkerd +33%, Cilium +99% under mTLS (arXiv:2411.02267)
- **W3C Trace Context Level 2** emerging standard for improved sampling-consistent trace ID randomness

## Quick Start

```bash
# Deploy with OpenTelemetry ingestion
deploy.yaml:
  backend: tempo
  storage: s3
  integrations:
    - otel_collector
    - cilium_hubble  # eBPF sidecar-free topology

# Configure predictive SLO breach detection
observability.yaml:
  predictive_breach:
    enabled: true
    lead_time: "15m"  # predict 15 minutes ahead
    signals:
      - memory_growth_rate
      - queue_depth
      - upstream_latency

# Natural language incident narration
# (automatically generated on alert)
# "The checkout service experienced a 340ms latency spike at 14:23 UTC.
#  Root cause: Connection pool exhaustion on the payment database.
#  Timeline: Batch job spawned 50 worker threads at 14:18, saturating
#  available DB connections. Recommended: increase pool size or add
#  rate limiting to the batch job."
```

## Target Users

1. **Platform/SRE Engineers** — Kubernetes observability at scale without per-host licensing
2. **Backend Engineering Managers** — debugging P95 latency across 30+ microservices
3. **FinOps / Cost-Conscious CTOs** — observability without $200+/host/month bills
4. **Enterprise Architects (Regulated)** — data residency, air-gapped deployment, audit trails
5. **Startups** — zero licensing cost with AI-powered incident response

## Related Standards

- OpenTelemetry (CNCF) — de-facto standard for distributed system instrumentation
- W3C Trace Context (Level 1 & 2) — standard HTTP header propagation for trace context
- OTLP (OpenTelemetry Protocol) — gRPC/HTTP/JSON wire protocol for telemetry export
- Prometheus Data Model / OpenMetrics — metrics exposition format
- eBPF (IETF research area) — kernel-level programmable observability

---

Built on research from CNCF/Grafana ecosystem, Cilium Hubble eBPF observability, and academic work on LLM-based root cause analysis in microservices. [Read the full research](./research.md) | [Feature roadmap](./features.md)
