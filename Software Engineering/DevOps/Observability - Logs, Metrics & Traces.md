---
title: "Observability - Logs, Metrics & Traces"
tags: [devops, devops/observability]
aliases: [Observability, Monitoring, OpenTelemetry, OTel, Prometheus, Grafana, SLO, SLI, Error Budget, RED method, USE method, Structured Logging, Distributed Tracing]
updated: 2026-09-27
---
# Observability - Logs, Metrics & Traces

<Note>
**TL;DR**

**Monitoring** tells you *whether* something is wrong (known failure modes); **observability** lets you ask *why*, including questions you did not anticipate. The classic signals are **metrics** (cheap aggregates, alerting), **logs** (detailed events) and **traces** (a request's path across services), now joined by **profiles**. **OpenTelemetry** is the vendor-neutral standard to produce and ship them; **Prometheus + Grafana** is the common open-source metrics stack. Alert on user-facing **SLOs** with **error budgets**, not on every CPU spike.
</Note>

## 1. The signals

| Signal | What | Good for | Cost profile | Tools |
|---|---|---|---|---|
| **Metrics** | Numeric time series with labels (`http_requests_total{route="/cart",code="500"}`) | Dashboards, alerting, trends | Cheap per event; **cardinality** is the cost driver | Prometheus, Mimir, Thanos, VictoriaMetrics, CloudWatch, Datadog |
| **Logs** | Timestamped events, ideally structured JSON | Debugging specifics, audit | Expensive at volume | Loki, Elasticsearch/OpenSearch, CloudWatch Logs, Splunk |
| **Traces** | Tree of **spans** for one request across services | Latency breakdown, dependencies, where errors start | Sample to control cost | Jaeger, Tempo, Zipkin, X-Ray, Honeycomb |
| **Profiles** | Sampled stack traces (CPU, memory) over time | Which code burns CPU/memory in prod | Low with eBPF | Pyroscope, Parca, OTel eBPF profiler |

They are most powerful **correlated**: a metric spike → exemplar trace ID → the trace → the logs with that `trace_id` → the profile of the slow function.

{/* diagram source: /diagrams/diagram-179.mmd */}
![diagram](/diagrams/diagram-179.svg)

## 2. Structured logging

```js
// ❌ unstructured
console.log(`User ${userId} failed to pay order ${orderId}: ${err.message}`);

// ✅ structured (pino); fields are queryable and trace-correlated
logger.error({
  event: "payment_failed",
  user_id: userId,
  order_id: orderId,
  amount_cents: 4599,
  error_code: err.code,
  trace_id: span.spanContext().traceId,
}, "payment failed");
```

Rules: log to **stdout** in JSON; consistent field names (OTel semantic conventions); levels used meaningfully (ERROR = someone should look); include request/trace IDs; **never log secrets or personal data** (mask tokens, emails, card numbers); sample noisy debug logs; set retention by value (30 days hot, archive cold).

## 3. Metrics and Prometheus

Metric types: **counter** (only goes up: requests, errors), **gauge** (up/down: queue depth, memory), **histogram** (bucketed distribution: latency, compute percentiles server-side), **summary** (client-side quantiles, cannot be aggregated across instances).

Prometheus **pulls** (scrapes) `/metrics` endpoints on an interval, stores time series locally, evaluates alert rules, and sends alerts to **Alertmanager** (grouping, silencing, routing to PagerDuty/Slack). Long-term, multi-cluster storage: Thanos, Mimir, VictoriaMetrics.

```promql
# Request rate per route (RED: Rate)
sum by (route) (rate(http_requests_total[5m]))

# Error ratio (RED: Errors)
sum(rate(http_requests_total{code=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# p99 latency from a histogram (RED: Duration)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# CPU saturation of containers vs their requests
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))
  / sum by (pod) (kube_pod_container_resource_requests{resource="cpu"})
```

<Warning>
**Cardinality**

Every unique label combination is a separate series. A label like `user_id` or raw URL path (`/orders/123`) creates millions of series and can take Prometheus down or explode a vendor bill. Use bounded labels (route templates, status class); put high-cardinality detail in traces and logs.
</Warning>

## 4. Traces and OpenTelemetry

A **trace** is a tree of **spans** (operation name, start/end, attributes, status, events, links). Context propagates between services in the W3C `traceparent` header.

**OpenTelemetry (OTel)** is a CNCF project providing: a **specification** and **semantic conventions** (standard attribute names like `http.request.method`, `db.system`), **APIs and SDKs** per language, **auto-instrumentation** (and the Kubernetes Operator to inject it), the **OTLP** wire protocol, and the **Collector** (receivers → processors → exporters) to route telemetry to any backend. Instrument once, switch vendors by changing Collector config.

```js
// Node.js: manual span around business logic (auto-instrumentation covers HTTP, DB, etc.)
import { trace, SpanStatusCode } from "@opentelemetry/api";
const tracer = trace.getTracer("checkout");

export async function placeOrder(cart) {
  return tracer.startActiveSpan("placeOrder", async (span) => {
    span.setAttribute("cart.items", cart.items.length);
    try {
      return await chargeAndSave(cart);
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

```yaml
# otel-collector.yaml (excerpt)
receivers:
  otlp: { protocols: { grpc: {}, http: {} } }
processors:
  memory_limiter: { check_interval: 1s, limit_percentage: 80 }
  batch: {}
  tail_sampling:          # keep all errors and slow traces, 10% of the rest
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow, type: latency, latency: { threshold_ms: 1000 } }
      - { name: rest, type: probabilistic, probabilistic: { sampling_percentage: 10 } }
exporters:
  otlphttp/tempo: { endpoint: http://tempo:4318 }
  prometheusremotewrite: { endpoint: http://mimir/api/v1/push }
service:
  pipelines:
    traces:  { receivers: [otlp], processors: [memory_limiter, tail_sampling, batch], exporters: [otlphttp/tempo] }
    metrics: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [prometheusremotewrite] }
```

**Sampling**: head sampling (decide at the start, cheap, may drop interesting traces) vs tail sampling (decide after the trace completes in the Collector, keeps errors/slow ones, needs memory and consistent routing).

## 5. What to measure: RED, USE and the Golden Signals

| Method | For | Signals |
|---|---|---|
| **RED** (Tom Wilkie) | Request-driven services | **R**ate, **E**rrors, **D**uration |
| **USE** (Brendan Gregg) | Resources (CPU, memory, disk, network, pools, queues) | **U**tilisation, **S**aturation, **E**rrors |
| **Four Golden Signals** (Google SRE) | User-facing systems | Latency, traffic, errors, saturation |

Dashboards: one per service with RED at the top, then dependencies and USE for its resources. Link dashboards to runbooks.

## 6. SLI, SLO, SLA and error budgets

- **SLI** (indicator): a measured ratio of good events, e.g. "proportion of `GET /api/*` requests served successfully in < 300 ms".
- **SLO** (objective): the target for an SLI over a window, e.g. **99.9% over 28 days**.
- **SLA** (agreement): a contractual promise with penalties, deliberately looser than the SLO.
- **Error budget** = 1 − SLO. 99.9% over 30 days ≈ **43 minutes** of full unavailability (or the equivalent in failed requests). Budget left → ship faster; budget burned → prioritise reliability.

| SLO | Allowed downtime / 30 days |
|---|---|
| 99% | 7.2 h |
| 99.9% | 43.2 min |
| 99.95% | 21.6 min |
| 99.99% | 4.3 min |

**Burn-rate alerting**: page when the budget is being consumed fast enough to matter, using multiple windows to be both fast and precise. E.g. page if the 1 h burn rate > 14.4 (2% of a 30-day budget in 1 h) **and** the 5 min burn rate > 14.4; open a ticket for slow burns (e.g. 6 h window > 6). This replaces dozens of threshold alerts with a few meaningful ones. Tools: Sloth, Pyrra, OpenSLO, vendor SLO features.

<Info>
**🆕 2025–2026**

- **OpenTelemetry Profiles entered public alpha (2026, announced March 2026)** as the fourth OTel signal: a unified profiling data format compatible with pprof, the **eBPF profiling agent** integrated with the Collector, and Kubernetes metadata enrichment. Not yet for critical production use. Traces and metrics are stable across major language SDKs; logs are stable in most.
- **Prometheus 3.0 (2024-11-14)** set the stage: native **OTLP ingestion**, UTF-8 metric/label names (OTel dotted names without mangling), a new UI, Remote Write 2.0. **Native histograms became a stable feature in Prometheus 3.8 (2025-12-02)**; Prometheus is at 3.15 (2026-09-25).
- Kubernetes v1.34 made **kubelet and API server tracing stable** (OTLP export), and v1.37 (2026-08) moved **native histograms** for Kubernetes component metrics to beta.
- OTel Collector releases continue monthly (v0.161.0 on 2026-09-16).
</Info>

## Pitfalls

<Warning>
**Common pitfalls**

- Alerting on causes (CPU 80%) instead of symptoms (users getting errors): noisy pages, missed incidents.
- Averages hiding pain: use percentiles (p95/p99) and histograms; averages of percentiles are meaningless.
- Unbounded label cardinality.
- Logs without trace IDs; traces without log correlation.
- Sampling away the errors you need (head sampling at 1%).
- 100% SLO targets: impossible and freezes delivery.
- Observability bill growing faster than traffic: set budgets, drop/aggregate in the Collector.
</Warning>

## Senior perspective

- Treat telemetry as a product: standard libraries, semantic conventions, default dashboards and SLO templates provided by the platform ([Platform Engineering & Internal Developer Platforms](/Software%20Engineering/DevOps/Platform%20Engineering%20%26%20Internal%20Developer%20Platforms)).
- Decide cost controls early: retention tiers, sampling, dropping debug logs at the Collector, metric cardinality limits per team.
- Instrument with OTel to stay vendor-neutral; negotiate vendors on ingest volume with real data.
- In design reviews ask: What are the SLIs? How will we know this is broken before customers tell us? How do we trace a request end-to-end across the queue?

## Interview questions

<Accordion title="Monitoring vs observability?">
Monitoring checks known failure modes with predefined dashboards and alerts. Observability is the ability to understand internal state from outputs well enough to debug unknown problems, typically with high-cardinality, correlated telemetry (traces, structured events).
</Accordion>

<Accordion title="Explain SLI, SLO, SLA and error budget.">
SLI is a measurement of good events/total; SLO is the target for it over a window; SLA is a contractual commitment with consequences (looser than the SLO); the error budget is 1 − SLO, the amount of unreliability allowed, used to balance feature velocity against reliability work.
</Accordion>

<Accordion title="RED vs USE?">
RED (Rate, Errors, Duration) describes request-driven services from the caller's view. USE (Utilisation, Saturation, Errors) describes resources. Use RED for services, USE for infrastructure and bottleneck hunting.
</Accordion>

<Accordion title="What is OpenTelemetry and why use it?">
A CNCF standard of APIs, SDKs, semantic conventions, the OTLP protocol and the Collector for generating and exporting traces, metrics, logs (and profiles). It decouples instrumentation from backends, avoiding vendor lock-in.
</Accordion>

<Accordion title="Head vs tail sampling?">
Head sampling decides at trace start (cheap, stateless, but random). Tail sampling decides after seeing the whole trace (keep errors and slow traces), requiring buffering in the Collector and routing all spans of a trace to the same instance.
</Accordion>

<Accordion title="Why is high cardinality a problem for metrics?">
Each label combination is a separate time series with memory and storage cost. Labels like user IDs create millions of series, slowing queries and exploding costs. Keep metric labels bounded; use traces/logs for per-request detail.
</Accordion>

<Accordion title="How would you alert on a 99.9% availability SLO?">
Multi-window, multi-burn-rate alerts: page on fast burn (e.g. 1 h and 5 min windows both above 14.4× burn), ticket on slow burn (e.g. 6 h/30 min above 6×). This catches big outages quickly and slow leaks without noise.
</Accordion>

## Related
- [DevOps Index](/Software%20Engineering/DevOps/DevOps%20Index) · [Site Reliability Engineering & Incident Management](/Software%20Engineering/DevOps/Site%20Reliability%20Engineering%20%26%20Incident%20Management) · [Kubernetes Workloads in Production](/Software%20Engineering/DevOps/Kubernetes/Kubernetes%20Workloads%20in%20Production) · [Microservices Architecture](/Software%20Engineering/DevOps/Microservices%20Architecture)
- [Core Web Vitals](/Software%20Engineering/Frontend/10%20Performance/Core%20Web%20Vitals) (frontend RUM side of observability) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
- Hands-on tooling: [Grafana & the LGTM Stack](/Software%20Engineering/DevOps/Delivery%20Platform/Grafana%20%26%20the%20LGTM%20Stack) · [Kubernetes Operations - Logs, Debugging & Scheduling](/Software%20Engineering/DevOps/Delivery%20Platform/Kubernetes%20Operations%20-%20Logs%2C%20Debugging%20%26%20Scheduling)

## Sources
- [OpenTelemetry docs: Concepts / Signals](https://opentelemetry.io/docs/concepts/signals/)
- [OpenTelemetry blog: Profiles enters public alpha (2026)](https://opentelemetry.io/blog/2026/profiles-alpha/)
- [Prometheus docs](https://prometheus.io/docs/introduction/overview/) · [Announcing Prometheus 3.0 (2024-11-14)](https://prometheus.io/blog/2024/11/14/prometheus-3-0/) · [Prometheus releases](https://github.com/prometheus/prometheus/releases)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) · [SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Brendan Gregg: The USE Method](https://www.brendangregg.com/usemethod.html)
- [Grafana: The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- Charity Majors, Liz Fong-Jones, George Miranda, *Observability Engineering* (O'Reilly, 2022)
