# OTLP/gRPC migration — plan and result

**Status: done and verified.** Every service pushes **traces, metrics and logs over OTLP/gRPC** to Alloy, and **profiles** to Alloy's Pyroscope receiver. After this, each Helm chart needs one monitoring value.

## Target contract (identical for all 10 services)

| Env var | Value | Meaning |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://alloy:4317` | gRPC endpoint (standard var; telemetry is on iff set). Legacy `COLLECTOR_SERVICE_ADDR` (`host:port`) still works as a fallback |
| `OTEL_SERVICE_NAME` | `<service>` | unchanged; dashboards depend on it |
| `OTEL_METRIC_EXPORT_INTERVAL` | `15000` (ms) | new, optional |
| `PYROSCOPE_ADDR` | `http://alloy:4040` | Alloy `pyroscope.receive_http` → Pyroscope |
| `ENABLE_PROFILING` | `1` | unchanged (Go services) |

Also added in every service: `service.instance.id` = pod hostname (without it, replicas overwrite each other's series in Mimir).

### What can and cannot be gRPC

| Signal | Result |
|---|---|
| Traces | OTLP/gRPC everywhere (Java, Node, Python moved from OTLP/HTTP; Go and .NET already used gRPC) |
| Metrics | OTLP/gRPC **push** replaces Prometheus scraping. Go: Prometheus registry bridged into OTel (names unchanged). Node/Python: exporter swap. Java: Micrometer → OTel SDK. .NET: `prometheus-net` → OTel meters (**names changed**) |
| Logs | OTLP/gRPC **in addition to** stdout (so `kubectl logs` still works). Log bridges add `trace_id`/`span_id`. Alloy no longer collects app stdout; only the DB and Redis, which cannot speak OTLP |
| Profiles | **Not OTLP** — the Pyroscope SDKs use their own HTTP push protocol. Routed through Alloy so apps only know one host. `cart-service` has no profiler |

## What changed, per service

| Service | Code changes |
|---|---|
| currency, payment (Node) | gRPC exporters for traces/metrics/logs, `PeriodicExportingMetricReader`, `instrumentation-pino`, `service.instance.id`; Prometheus exporter and `METRICS_PORT` removed; **event-loop lag probe rewritten** (the old `setTimeout(0)` loop pinned an idle pod at ~25 % CPU); `BatchSpanProcessor` instead of `SimpleSpanProcessor` |
| email, recommendation (Python) | `telemetry.py` rewritten for gRPC exporters + `LoggingHandler`; `requirements.in` / `requirements.txt` regenerated; `prometheus_client` and the Prometheus reader removed; email's test stubs updated for the new module set |
| product-catalog, shipping, checkout, frontend (Go) | new `cmd/server/otlp.go` (`initTelemetry`): `otlptracegrpc`, Prometheus→OTel bridge → `otlpmetricgrpc`, `otellogrus` → `otlploggrpc`; old `initTracing` removed; package loggers of `checkout` and `catalog` exposed via `Logger()` so they are hooked too; `go.mod` pinned to the OTel versions that match Go 1.25 (newer ones need Go 1.26) |
| ad (Java) | `Telemetry.init()` builds traces + metrics + logs SDK with `OtlpGrpc*Exporter`; `Metrics.java` on OTel instruments (Micrometer removed); Log4j2 `OpenTelemetryAppender`; metrics HTTP server and Docker `HEALTHCHECK`/`EXPOSE 9464` removed; test updated |
| cart (.NET) | `Program.cs` on OpenTelemetry traces/metrics/logs over OTLP/gRPC (`prometheus-net` removed, Kestrel metrics endpoint removed); OpenTelemetry packages bumped to **1.15.3** (security advisories broke the build under `TreatWarningsAsErrors`); root `.dockerignore` added |

Existing unit tests were kept and re-run: currency 18, payment 20, email 37, recommendation 19 (pytest/jest), Go `go vet` + `go test` for all four Go services, ad `gradle test`; cart builds clean.

## Observability repo

- `alloy/config.otlp.alloy` — OTLP/gRPC receiver → Tempo / Mimir (via `otelcol.exporter.prometheus`) / Loki (native `/otlp` endpoint); Pyroscope receiver; stdout collection only for `product-catalog-db` and `cart-redis`.
- `docker-compose.otlp.yaml` — overlay: `docker compose -f docker-compose.yaml -f docker-compose.otlp.yaml up -d`.
- Dashboards: `currency.json` and `payment.json` (`rpc_server_duration_seconds_*`), `cart.json` (rewritten onto `http_server_*` metrics and `dotnet_*`). The other dashboards are unchanged.
- Originals were left in place otherwise; the old `config.alloy` still works with the old images.

## How it was verified

1. A test stack (Alloy, Mimir, Loki, Tempo, Pyroscope + all ten services + Postgres + Redis) with every app container **read-only, non-root, all capabilities dropped**.
2. Real traffic through the frontend: browse, add to cart, checkout (25 + 16 sessions).
3. Checked per service: series in Mimir (`job`, `instance`), logs in Loki (`service_name`, `trace_id`), traces in Tempo (all ten services), profiles in Pyroscope (all but cart).
4. All 198 dashboard queries executed against the stack: 186 return data; the 12 empty ones are error-rate / status-code panels (no errors occurred).
5. The compose overlay in the observability repo was started separately and checked the same way.
6. The same images ran in a kind cluster through the Helm charts (see `platform-demo-helm/README.md`).

## Things you need to do

- **Push the images.** _Done (2026-10-17):_ `CI - Main` built and pushed them to Docker Hub (signed, with SBOM), their digests are pinned in the charts, and `v1.0.0` publishes `:1.0.0` for each service. The local `:1.1.0` tags were never pushed and do not exist on Docker Hub.
- The old compose file still pins the **pre-OTLP** digests and keeps working with `config.alloy`; use the overlay for the new setup.
- Nothing was committed to any git repository.

## Known follow-ups (not done)

- `payment-service` still logs the full request including card data (now also over OTLP).
- `SIGTERM` handling in Node/Python; JSON logs for cart; `postgres-exporter`.
- `cart-service` gRPC server meter (`Grpc.AspNetCore.Server`) emitted nothing; per-method numbers come from `http_route` on the ASP.NET Core metrics instead.
