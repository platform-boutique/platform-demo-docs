# Observability contract

All ten services now push **traces, metrics and logs over OTLP/gRPC** to Alloy and send **profiles** to Alloy's Pyroscope receiver. Nothing is scraped, and application containers are no
longer collected from stdout (`kubectl logs` still works — stdout output was kept). What changed and why is in [`OTLP-MIGRATION-PLAN.md`](https://github.com/platform-boutique/platform-demo-docs/blob/main/OTLP-MIGRATION-PLAN.md).

> Profiles cannot use OTLP: the Pyroscope SDKs push their own HTTP protocol. They are sent to Alloy (`pyroscope.receive_http` on `:4040`), which forwards to Pyroscope, so applications still only know one host.

### 6.1 The contract (identical for every service)

| Env var | Value | Purpose |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://alloy.monitoring.svc.cluster.local:4317` (compose: `http://alloy:4317`) | One OTLP/**gRPC** endpoint for traces, metrics and logs (scheme required). Legacy fallback: `COLLECTOR_SERVICE_ADDR` (`host:port`) |
| `OTEL_SERVICE_NAME` | the service name | becomes Prometheus `job` and Loki `service_name` — do not change |
| `OTEL_METRIC_EXPORT_INTERVAL` | `15000` (ms) | metric push interval |
| `PYROSCOPE_ADDR` | `http://alloy.monitoring.svc.cluster.local:4040` | profiles (not for `cart-service`, which has no profiler) |
| `ENABLE_PROFILING` | `1` | Go services only |
| `OTEL_RESOURCE_ATTRIBUTES` | `k8s.pod.name=…,k8s.namespace.name=…,k8s.node.name=…` | injected by the Helm charts via the downward API |

`service.instance.id` is set in code from the pod hostname (`HOSTNAME`), and becomes the Prometheus `instance` label. It is mandatory: without it two replicas export identical series and overwrite each other.

Telemetry is on iff an endpoint is available (Go services: no endpoint = off; `ENABLE_TRACING=0` forces it off). With the collector down, services keep serving and only log exporter errors — verified.

### 6.2 How each stack implements it

| Service(s) | Traces | Metrics | Logs |
|---|---|---|---|
| Go: frontend, checkout, product-catalog, shipping | `otlptracegrpc` + `otelgrpc`/`otelhttp` | existing Prometheus registry **bridged** into OTel (`contrib/bridges/prometheus`) → `otlpmetricgrpc`; metric names unchanged | `logrus` hook (`otellogrus`) → `otlploggrpc`; stdout JSON kept |
| Node: currency, payment | `NodeSDK` + OTLP gRPC exporter | hand-rolled instruments + runtime gauges → `PeriodicExportingMetricReader` (OTLP gRPC) | `pino` + `@opentelemetry/instrumentation-pino` → OTLP gRPC |
| Python: email, recommendation | OTLP gRPC span exporter | hand-rolled instruments → OTLP gRPC | `LoggingHandler` → OTLP gRPC |
| Java: ad | `OtlpGrpcSpanExporter` | OTel SDK instruments (replaced Micrometer) → `OtlpGrpcMetricExporter` | Log4j2 `OpenTelemetryAppender` → `OtlpGrpcLogRecordExporter` |
| .NET: cart | OTel .NET OTLP gRPC | OTel ASP.NET Core / Kestrel / runtime meters (replaced `prometheus-net`; **names changed**) | OTel logging provider → OTLP gRPC |

Profiles: Pyroscope SDK in every service except cart (Go SDK, `@pyroscope/nodejs`, `pyroscope-io`, Java agent). The Java agent extracts a native library into `/tmp`, so `/tmp` must be writable **and executable**.

### 6.3 What the Grafana dashboards depend on

- **Metrics:** `job="<service name>"` and `instance=<pod>`. **Logs (Loki):** `service_name="<service name>"` (plus `trace_id`, `span_id`, `detected_level`).
- **Metric families** (dashboards updated where marked):
  - Go gRPC (checkout, product-catalog, shipping): `grpc_server_handled_total`, `grpc_server_handling_seconds_*`, `grpc_server_started_total` — unchanged
  - ad, email, recommendation, **currency\***, **payment\***: `rpc_server_duration_seconds_{bucket,count,sum}`, `rpc_server_requests_total`, `rpc_server_active_requests` (\*Node services gained the `_seconds` suffix; dashboards updated)
  - **cart\***: `http_server_request_duration_seconds_*` (label `http_route="/hipstershop.CartService/<Method>"`), `http_server_active_requests`, `dotnet_*`, `kestrel_*` (dashboard rewritten)
  - frontend: `http_requests_total`, `http_request_duration_seconds_*`, `http_requests_in_flight` — unchanged
- Verified: all 198 dashboard queries were run against the live test stack — 186 return data; the other 12 are error-rate / status-code panels that are empty simply because no errors occurred.

### 6.4 Backends and the Alloy pipeline

| Backend | Used for | Compose address |
|---|---|---|
| Alloy | OTLP/gRPC receiver `:4317`, Pyroscope receiver `:4040`, UI `:12345` | `alloy` |
| Mimir | metrics (remote_write `/api/v1/push`, query `/prometheus`) | `mimir:9009` |
| Loki | logs, via its **native OTLP endpoint** `http://loki:3100/otlp` (needs Loki ≥ 3; `service.name` → `service_name`) | `loki:3100` |
| Tempo | traces (OTLP gRPC `tempo:4317`, query `:3200`) | `tempo` |
| Pyroscope | profiles | `pyroscope:4040` |
| Grafana | UI, datasources uids `mimir`, `loki`, `tempo`, `pyroscope` | `grafana:3000` |

The pipeline is [`platform-demo-observability/alloy/config.otlp.alloy`](https://github.com/platform-boutique/platform-demo-observability/blob/main/alloy/config.otlp.alloy): `otelcol.receiver.otlp` (gRPC) → `otelcol.processor.batch` →
traces: `otelcol.exporter.otlp` (Tempo) · metrics: `otelcol.exporter.prometheus` → `prometheus.remote_write` (Mimir) · logs: `otelcol.exporter.otlphttp` (Loki `/otlp`);
`pyroscope.receive_http` → `pyroscope.write`; and `loki.source.docker` **only** for `product-catalog-db` and `cart-redis` (they cannot speak OTLP).

### 6.5 Kubernetes notes for Alloy

- The Alloy `Service` must expose `4317` (OTLP/gRPC) and `4040` (Pyroscope). The charts' NetworkPolicies allow egress to exactly those ports in the namespace `global.monitoring.namespace` (default `monitoring`).
- Re-point the exporters in `config.otlp.alloy` at the in-cluster Mimir/Loki/Tempo/Pyroscope Services of whichever monitoring charts you install (`monitoring/` holds vendored upstream charts).
- Collect the two non-OTLP workloads from pod logs, e.g. with `discovery.kubernetes` + `discovery.relabel` keeping pods whose label `app.kubernetes.io/component` is `database` or `cache` (the charts set it), feeding `loki.source.kubernetes` and setting `service_name` from the pod label `app.kubernetes.io/name`. This is implemented and was exercised end to end in `platform-demo-helm/monitoring-local` (single-replica Alloy with Kubernetes pod discovery tailing the database and cache logs); see §8.8.
- Both new rules need nothing else from the cluster: no scrape annotations, no `ServiceMonitor`.
