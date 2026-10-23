# platform-demo — master reference

A polyglot e-commerce microservices application (an Online-Boutique-style shop) built as the workload for a full platform-engineering project: observability (Grafana LGTM + Alloy), GitOps (Argo CD), policy (Kyverno), infrastructure provisioning (Crossplane) and a developer portal (Backstage).

**This document is the single source of truth for the application as a whole.** It is derived from the source code, Dockerfiles, the observability compose stack and the built images (not from the old READMEs) and is meant to be pasted into a chat/agent session while building the Kubernetes deployment — Helm chart by Helm chart. Each service's own `README.md` has the per-service detail (config tables, probes, security context, values skeleton); this file holds everything cross-cutting.

_Last verified against the repositories on 2026-10-17 (end of the Kyverno session; before it, the Argo CD / Cilium session), **after the OTLP/gRPC migration** (see [`OTLP-MIGRATION-PLAN.md`](OTLP-MIGRATION-PLAN.md)) and after everything was promoted to `main`, put behind the shared CI, branch protection and `v1.0.0` tags, deployed by Argo CD on a Cilium cluster (§13, §8.8) and guarded by Kyverno admission policies (§13). The first analysis ignored `tests/`, `__tests__/`, `ci/` and `.github/`; the migration only touched the tests that stub the telemetry modules._

---

## Contents

1. [System at a glance](#1-system-at-a-glance)
2. [Architecture & request flows](#2-architecture--request-flows)
3. [Service inventory](#3-service-inventory)
4. [Container images](#4-container-images)
5. [Service wiring (who calls whom, with which env var)](#5-service-wiring)
6. [Observability contract](#6-observability-contract)
7. [Kubernetes translation guide](#7-kubernetes-translation-guide)
8. [Helm charts](#8-helm-charts)
9. [Cross-cutting findings & gotchas](#9-cross-cutting-findings--gotchas)
10. [Local stack (docker compose)](#10-local-stack-docker-compose)
11. [Repository map](#11-repository-map)
12. [Next steps](#12-next-steps)
13. [CI/CD, branching and releases](#13-cicd-branching-and-releases)

---

## 1. System at a glance

| Item | Value |
|---|---|
| Application services | 10 (`frontend` + 9 backends) |
| Data stores | PostgreSQL 17 (`product-catalog-db`), Redis 7 (`cart-redis`) |
| Inter-service protocol | gRPC over plaintext HTTP/2 (no TLS, no auth, no service mesh) |
| External protocol | HTTP/1.1 from `frontend` only |
| Languages | Go (frontend, checkout, product-catalog, shipping), Java 21 (ad), C#/.NET 10 (cart), TypeScript/Node 22 (currency, payment), Python 3.12 (email, recommendation) |
| Registry | Docker Hub, `mladenovskistefan/<service>` |
| Optional / dormant | `shopping-assistant-service` (Python, GCP-only, not wired in) |
| Observability | Services **push** traces + metrics + logs over **OTLP/gRPC** to Alloy → Tempo / Mimir / Loki; profiles go to Alloy → Pyroscope. Grafana on top |
| Orchestration | `docker compose` (+ `docker-compose.otlp.yaml` overlay) in `platform-demo-observability/`; production-grade **Helm charts** in `platform-demo-helm/` (§8) |

Services are stateless except the two data stores. Only `frontend` needs to be reachable from outside the cluster.

---

## 2. Architecture & request flows

```mermaid
flowchart LR
    U([Browser]) -->|HTTP :8080| FE[frontend]

    FE -->|gRPC :3550| PC[product-catalog-service]
    FE -->|gRPC :7000| CUR[currency-service]
    FE -->|gRPC :7070| CART[cart-service]
    FE -->|gRPC :8080| REC[recommendation-service]
    FE -->|gRPC :5050| CHK[checkout-service]
    FE -->|gRPC :50051| SHIP[shipping-service]
    FE -->|gRPC :9555| AD[ad-service]

    CHK -->|:3550| PC
    CHK -->|:7070| CART
    CHK -->|:7000| CUR
    CHK -->|:50051| SHIP
    CHK -->|:50051| PAY[payment-service]
    CHK -->|:8080| EML[email-service]

    REC -->|:3550| PC
    PC -->|SQL :5432| DB[(product-catalog-db<br/>PostgreSQL)]
    CART -->|RESP :6379| RDS[(cart-redis<br/>Redis)]

    subgraph Observability
      ALLOY[Alloy] --> MIMIR[Mimir]
      ALLOY --> LOKI[Loki]
      ALLOY --> TEMPO[Tempo]
      ALLOY --> PYRO[Pyroscope]
      GRAF[Grafana] -.queries.-> MIMIR & LOKI & TEMPO & PYRO
    end
```

Leaf services (no outbound calls): `ad`, `currency`, `payment`, `email`, `shipping`. Data stores sit behind `product-catalog-service` and `cart-service` only — nothing else talks to them.

### Browse / add to cart

1. Browser → `frontend` `GET /`. The frontend sets a `shop_session-id` cookie (random UUID, used as the cart `user_id`).
2. In parallel: `currency.GetSupportedCurrencies`, `product-catalog.ListProducts`, `cart.GetCart`; then `currency.Convert` per product; `ad.GetAds` (100 ms deadline, failure ignored).
3. Product page adds `recommendation.ListRecommendations` → `product-catalog.ListProducts` (inside the recommendation service) → up to 4 `product-catalog.GetProduct`.
4. "Add to cart" → `POST /cart` → `cart.AddItem` (Redis write).

### Checkout

`POST /cart/checkout` → `frontend` validates the form → `checkout.PlaceOrder`:

1. `cart.GetCart` → per item (parallel) `product-catalog.GetProduct` + `currency.Convert` → `shipping.GetQuote` + `currency.Convert`
2. `payment.Charge` (fail → abort)
3. `shipping.ShipOrder` (fail → error; card already charged — **no rollback**)
4. `cart.EmptyCart` (error ignored)
5. `email.SendOrderConfirmation` (error logged, order still succeeds)

Trace context (W3C `traceparent`) is generated at the frontend and propagated through every gRPC hop, so one checkout produces one trace spanning up to 8 services.

---

## 3. Service inventory

### 3.1 Runtime facts

| Service | Dir | Lang / runtime | gRPC / HTTP port | Metrics (pushed via OTLP) | Workload kind | Stateful | Runs as (verified) | Arch |
|---|---|---|---|---|---|---|---|---|
| frontend | `frontend` | Go 1.25 | **8080** HTTP | — (debug `/metrics` 9464) | Deployment | no | `appuser` UID 100 / GID 101 | builder arch |
| ad-service | `ad-service` | Java 21 | **9555** gRPC | — | Deployment | no | `appuser` UID 100 / GID 101 | builder arch |
| cart-service | `cart-service` | .NET 10 | **7070** gRPC | — | Deployment | no (Redis) | `app` UID/GID **1654** | builder arch |
| checkout-service | `checkout-service` | Go 1.25 | **5050** gRPC | — (debug `/metrics` 9090) | Deployment | no | **root (UID 0)** | **amd64 only** |
| currency-service | `currency-service` | Node 22 / TS | **7000** gRPC | — | Deployment | no | `appuser` UID 100 / GID 101 | builder arch |
| email-service | `email-service` | Python 3.12 | **8080** gRPC | — | Deployment | no | `appuser` UID 100 / GID 101 | builder arch |
| payment-service | `payment-service` | Node 22 / TS | **50051** gRPC | — | Deployment | no | `appuser` UID 100 / GID 101 | builder arch |
| product-catalog-service | `product-catalog-service` | Go 1.25 | **3550** gRPC | — (debug `/metrics` 9090) | Deployment | no (Postgres, in-mem cache) | **root (UID 0)** | **amd64 only** |
| recommendation-service | `recommendation-service` | Python 3.12 | **8080** gRPC | — | Deployment | no | `appuser` UID 100 / GID 101 | builder arch |
| shipping-service | `shipping-service` | Go 1.25 | **50051** gRPC | — (debug `/metrics` + pprof 9090) | Deployment | no | **root (UID 0)** | **amd64 only** |
| product-catalog-db | `product-catalog-db` | PostgreSQL 17 | **5432** | — | **StatefulSet** (1 replica) | **yes**, PVC | `postgres` UID/GID **999** | image arch |
| cart-redis | (no repo; `redis:7-alpine`) | Redis 7 | **6379** | — | **StatefulSet** (1 replica) | yes (cache; PVC optional) | `redis` (image default) | image arch |
| shopping-assistant-service | `shopping-assistant-service` | Python 3.14 / Flask | 8080 HTTP | none | — (**not deployed**) | no | root | — |

Port collisions are harmless in Kubernetes (separate pods/Services): `8080` is used by frontend, email and recommendation; `50051` by shipping and payment. Since the OTLP migration only the Go services still serve a (debug-only) Prometheus `/metrics`; nothing scrapes it.

### 3.2 Roles & statefulness

| Service | One-line role | Notes |
|---|---|---|
| frontend | storefront HTML + session cookie, fans out to 7 backends | needs HTTPS (Secure cookies) |
| ad-service | contextual ads from a hard-coded catalog | tracing/profiling always on |
| cart-service | per-user cart in Redis | health check pings Redis |
| checkout-service | orchestrates `PlaceOrder` across 6 services | no compensation on failure |
| currency-service | converts via static EUR-based rates (33 currencies) | rates JSON baked in the image |
| email-service | **stub** — renders a template and logs; sends nothing | |
| payment-service | **simulated** card validation (Visa/MasterCard, expiry) | logs raw card data |
| product-catalog-service | product reads, loaded from Postgres into memory | **exits if DB unreachable at start** |
| recommendation-service | random picks from the catalog excluding what the user has | needs `product-catalog` address at start |
| shipping-service | flat $8.99 quote + fake tracking ID | pprof always exposed |
| product-catalog-db | Postgres with schema + 9 products baked in | init script runs only on empty data dir |

---

## 4. Container images

Repository names are `mladenovskistefan/<dir-name>`. Images are built, Trivy-scanned, cosign-signed (keyless) and published to Docker Hub by each service's `release.yaml` (§13): a `v*` tag publishes `:<version>`, `:sha-<short>` and `:latest`. `v1.0.0` was tagged on 2026-10-17, so `:1.0.0` exists for the ten services.

The environments pin **`image.digest`** in `platform-demo-helm/envs/<env>/values/<app>.yaml` (Argo CD deploys exactly these). The `dev` environment runs the **`v1.0.0`** release images:

| Image | Digest deployed by `envs/dev` (`v1.0.0`) |
|---|---|
| frontend | `mladenovskistefan/frontend@sha256:fc355592e4a67d4c10898297f1b5d38422b2859fa5373341fdb2bce857d66715` |
| product-catalog-service | `mladenovskistefan/product-catalog-service@sha256:22c843ec2cc73b20b3a7226877c5b08c141cb0a65e28c02d4257a0db15acb141` |
| checkout-service | `mladenovskistefan/checkout-service@sha256:9d05fb52e2ab1184b45680e1414bddf8cc06e37e55d78cc76b0b652a49879c49` |
| cart-service | `mladenovskistefan/cart-service@sha256:878ad0cf03353ac2f0e8b41edb98443381be561a9a850e7a6e0403e1b0e384ea` |
| currency-service | `mladenovskistefan/currency-service@sha256:d8c4b657e67128c15ae6d2280beef27e81e6374e94a76177a06a48cb0f2911e4` |
| shipping-service | `mladenovskistefan/shipping-service@sha256:cc10d4c55a92c76687b06113caaa07b5c5e6a841d0c5bc046e00cd3ed04d5976` |
| payment-service | `mladenovskistefan/payment-service@sha256:c292f5af4944adbb528e273b9e6822d9890132f56e26301b88bd7c7e27770af4` |
| email-service | `mladenovskistefan/email-service@sha256:e495b8e67397daddf5803037d633e825c298c2a189eb7dabf8bf4afcdb553808` |
| recommendation-service | `mladenovskistefan/recommendation-service@sha256:2ccac2cddeb4ff3759adbfe96e628379bcadb2fed82f201171ae134de286390f` |
| ad-service | `mladenovskistefan/ad-service@sha256:dc106726ade580e7a2ba16d9d39270d89f7afca59809d2667ff1add384a1ce64` |
| product-catalog-db | `mladenovskistefan/product-catalog-db@sha256:c1edd3a805e892df3edee101c046f8a7e45ab38ab670e981cb1a91a4ce3b355b` (chart default; temporary local-testing image) |
| cart-redis | `redis@sha256:8b81dd37ff027bec4e516d41acfbe9fe2460070dc6d4a4570a2ac5b9d59df065` (upstream `redis:7-alpine`, chart default) |

The chart defaults (`<chart>/values.yaml`) still contain the digests of the earlier OTLP builds from the `CI - Main` runs of 2026-10-16/17 (same code, different digests); environments override them. Promotion = a PR that changes the digest in `envs/<env>/values/<app>.yaml`.

- The Helm repo's pull-request pipeline verifies that every `mladenovskistefan/*` image in the rendered charts is pinned by digest **and** carries a cosign signature from one of our pipelines.
- `product-catalog-db` is a **temporary, local-testing image** (Postgres + schema + seed), built by hand, with no pipeline and no signature. When the app is deployed on AWS it is replaced by **RDS** and the image, its chart and its allow-list entry in the Helm CI are deleted.
- `platform-demo-observability/docker-compose.yaml` (the original compose file) still pins the **pre-OTLP** digests, which use the old env contract.

### 4.1 How the images are built (from the Dockerfiles)

All are multi-stage, built with the **service root as build context**.

| Service | Builder base | Runtime base | Build step | Entrypoint | EXPOSE | Docker HEALTHCHECK |
|---|---|---|---|---|---|---|
| frontend | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` | `./frontend` (cwd `/app`) | 8080 | `wget /_healthz` on `${PORT:-8080}` |
| ad-service | `eclipse-temurin:21-jdk-alpine` | `eclipse-temurin:21-jre-alpine` | `./gradlew installDist` | `bin/AdService` | 9555 | none |
| cart-service | `mcr.microsoft.com/dotnet/sdk:10.0` | `mcr.microsoft.com/dotnet/aspnet:10.0` | `dotnet publish -c Release` | `./cartservice` | 7070 | none |
| checkout-service | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` (`GOARCH=amd64`) | `/app/checkout-service` | 5050, 9090 | none |
| currency-service | `node:22.22.2-alpine3.23` | same | `npm ci` + `tsc` + `npm prune --omit=dev` | `node dist/server.js` | 7000 | grpc-js client to `localhost:$PORT` (**needs `PORT` set**) |
| email-service | `python:3.12-alpine` | same | `pip install requirements.in` + `protoc` | `python src/server.py` | 8080 | `channel_ready_future` on `${PORT:-8080}` |
| payment-service | `node:22.22.2-alpine3.23` | same | as currency | `node dist/server.js` | 50051 | grpc-js client to `localhost:$PORT` (**needs `PORT` set**) |
| product-catalog-service | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` (`GOARCH=amd64`) | `/app/product-catalog-service` | 3550, 9090 | none |
| recommendation-service | `python:3.12-alpine` | same | as email | `python src/server.py` | 8080 | `channel_ready_future` on `${PORT:-8080}` |
| shipping-service | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` (`GOARCH=amd64`) | `/app/shipping-service` | 50051, 9090 | none |
| product-catalog-db | — | `postgres:17` | copies `dump.sql` to `/docker-entrypoint-initdb.d/001_init.sql` | stock postgres entrypoint | 5432 | none |

Docker `HEALTHCHECK` instructions are **ignored by Kubernetes** — define probes in the chart (§7.3).

Build notes:

- Python services generate their gRPC stubs **inside the Docker build** (stubs are not committed) and install from `requirements.in` (transitive deps float).
- Go services embed generated protobuf code in the repo (`proto/`, `genproto/`).
- `cart-service` now has a root `.dockerignore` (the old one in `src/` had no effect, and a host-built `src/obj` broke the in-image publish with `NETSDK1064`).
- Go services `checkout`, `product-catalog`, `shipping` hard-code `GOARCH=amd64`; the three are the ones that run as root.
- The Node services' runtime stage runs `apk upgrade --no-cache` and removes npm/npx (their bundled `tar`, `pacote`, `sigstore` otherwise trip Trivy). Both keep a `.trivyignore` that should be re-reviewed (`CVE-2026-22184`, `CVE-2026-33671`).
- Base images are pinned by minor tag only (no digests); Dependabot proposes bumps (§13).

---

## 5. Service wiring

### 5.1 Dependency matrix (caller → callee)

| Caller ↓ / Callee → | product-catalog | currency | cart | recommendation | checkout | shipping | payment | email | ad | postgres | redis |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **frontend** | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | | | ✔ | | |
| **checkout** | ✔ | ✔ | ✔ | | | ✔ | ✔ | ✔ | | | |
| **recommendation** | ✔ | | | | | | | | | | |
| **product-catalog** | | | | | | | | | | ✔ | |
| **cart** | | | | | | | | | | | ✔ |
| ad / currency / payment / email / shipping | — leaf services, no outbound calls — |

### 5.2 Required environment per service (values used by compose; Kubernetes uses Service DNS names)

Address variables are **`host:port` with no scheme**. All hostnames below are the compose service names — keep the Kubernetes `Service` names identical so these values keep working inside one namespace, or use FQDNs (`<svc>.<ns>.svc.cluster.local`) across namespaces.

| Service | Variable | Compose value | Required? | Behaviour if missing |
|---|---|---|---|---|
| frontend | `PRODUCT_CATALOG_SERVICE_ADDR` | `product-catalog-service:3550` | **yes** | panic at start |
| frontend | `CURRENCY_SERVICE_ADDR` | `currency-service:7000` | **yes** | panic |
| frontend | `CART_SERVICE_ADDR` | `cart-service:7070` | **yes** | panic |
| frontend | `RECOMMENDATION_SERVICE_ADDR` | `recommendation-service:8080` | **yes** | panic |
| frontend | `CHECKOUT_SERVICE_ADDR` | `checkout-service:5050` | **yes** | panic |
| frontend | `SHIPPING_SERVICE_ADDR` | `shipping-service:50051` | **yes** | panic |
| frontend | `AD_SERVICE_ADDR` | `ad-service:9555` | **yes** | panic |
| checkout | `PRODUCT_CATALOG_SERVICE_ADDR` | `product-catalog-service:3550` | no (same default) | falls back to default |
| checkout | `CART_SERVICE_ADDR` | `cart-service:7070` | no | default |
| checkout | `CURRENCY_SERVICE_ADDR` | `currency-service:7000` | no | default |
| checkout | `SHIPPING_SERVICE_ADDR` | `shipping-service:50051` | no | default |
| checkout | `PAYMENT_SERVICE_ADDR` | `payment-service:50051` | no | default |
| checkout | `EMAIL_SERVICE_ADDR` | `email-service:8080` | no | default |
| recommendation | `PRODUCT_CATALOG_SERVICE_ADDR` | `product-catalog-service:3550` | **yes** | `RuntimeError` at start |
| product-catalog | `DATABASE_URL` | `postgres://catalog:catalog@product-catalog-db:5432/productcatalog` | **yes** | fatal at start (also fatal if DB unreachable) |
| cart | `REDIS_ADDR` | `cart-redis:6379` | **yes** | exception at start |

Because the defaults in `checkout` equal the compose values, `checkout` works with zero address config *only if* every service is in the same namespace under these exact Service names.

### 5.3 Secrets & sensitive values

| Value | Where used | Today | In Kubernetes |
|---|---|---|---|
| Postgres `POSTGRES_USER/PASSWORD/DB` | `product-catalog-db` | `catalog` / `catalog` / `productcatalog` (plain in compose) | `Secret`, `envFrom` |
| `DATABASE_URL` (embeds the password) | `product-catalog-service` | plain env | `Secret` + `secretKeyRef` |
| Redis password (if added) | `cart-service` `REDIS_ADDR` (`host:port,password=…`) | none (no auth) | `Secret` |
| Credit-card data in transit | `frontend` → `checkout` → `payment` | test cards only; **payment logs the full request** | redact in code before real use; NetworkPolicy |
| Google credentials / API key | `shopping-assistant-service` | not deployed | `Secret` + Workload Identity |

No other credentials exist. gRPC traffic is unauthenticated and unencrypted.

---

## 6. Observability contract

All ten services now push **traces, metrics and logs over OTLP/gRPC** to Alloy and send **profiles** to Alloy's Pyroscope receiver. Nothing is scraped, and application containers are no
longer collected from stdout (`kubectl logs` still works — stdout output was kept). What changed and why is in [`OTLP-MIGRATION-PLAN.md`](OTLP-MIGRATION-PLAN.md).

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

## 7. Kubernetes translation guide

### 7.1 Namespaces & DNS

Two workable layouts — pick one and keep the env vars consistent:

| Layout | DNS the apps must use | Trade-off |
|---|---|---|
| **A. One app namespace** (e.g. `platform-demo`) + `monitoring` | short names (`cart-service:7070`) work as-is | simplest; compose values and `checkout` defaults need no change |
| **B. Namespace per service** (the earlier Helm iteration did this) | FQDNs everywhere: `cart-service.cart-service.svc.cluster.local:7070` | per-team isolation, but every address in §5.2 must become an FQDN and NetworkPolicies/scrape config get more verbose |

Kubernetes `Service` names must equal the compose names (`frontend`, `ad-service`, `cart-service`, `checkout-service`, `currency-service`, `email-service`, `payment-service`, `product-catalog-service`, `recommendation-service`, `shipping-service`, `product-catalog-db`, `cart-redis`).

### 7.2 Ports to declare (container → Service)

Only one port per workload needs to be declared now that metrics are pushed (the charts name it `grpc`/`http`/`postgres`/`redis`). The Go services additionally serve a debug-only `/metrics` that the charts do not expose.

| Service | Container port | Service port |
|---|---|---|
| frontend | 8080 | 80 |
| ad-service | 9555 | 9555 |
| cart-service | 7070 | 7070 |
| checkout-service | 5050 | 5050 |
| currency-service | 7000 | 7000 |
| email-service | 8080 | 8080 |
| payment-service | 50051 | 50051 |
| product-catalog-service | 3550 | 3550 |
| recommendation-service | 8080 | 8080 |
| shipping-service | 50051 | 50051 |
| product-catalog-db | 5432 | 5432 |
| cart-redis | 6379 | 6379 |

### 7.3 Probes

Kubernetes native `grpc:` probes work for every gRPC service because each implements `grpc.health.v1.Health/Check` on its main port. All checks except cart's are **static** (`SERVING`) — they prove the process is alive, not that its dependencies are.

| Service | Readiness | Liveness | Startup | Note |
|---|---|---|---|---|
| frontend | `httpGet /_healthz :8080` | same | optional (instant) | static `ok` |
| ad-service | `grpc :9555` | `grpc :9555` | **recommended** (JVM + Pyroscope agent; ≥ 15 s window) | |
| cart-service | `grpc :7070` (**pings Redis**) | **`tcpSocket :7070`** (do not tie liveness to Redis) | `tcpSocket` | readiness drops while Redis is down |
| checkout-service | `grpc :5050` | `grpc :5050` | none | |
| currency-service | `grpc :7000` | `grpc :7000` | none | |
| email-service | `grpc :8080` | `grpc :8080` | none | |
| payment-service | `grpc :50051` | `grpc :50051` | none | |
| product-catalog-service | `grpc :3550` | `grpc :3550` | none | process only starts serving after the DB load succeeds |
| recommendation-service | `grpc :8080` | `grpc :8080` | none | |
| shipping-service | `grpc :50051` | `grpc :50051` | none | |
| product-catalog-db | `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB` | same (delay 30 s) | — | exec probe |
| cart-redis | `redis-cli ping` | same | — | exec probe |

Typical timing: `periodSeconds: 5` (readiness), `10` (liveness), `failureThreshold: 3`.

### 7.4 Startup order & dependency behaviour

Compose `depends_on` ordering does **not** exist in Kubernetes. What each service actually does when a dependency is missing:

| Service | Dependency missing at start | Handling in the chart |
|---|---|---|
| product-catalog-service | **exits non-zero** if Postgres is unreachable | init container `pg_isready` loop (snippet in its README), or accept `CrashLoopBackOff` |
| cart-service | starts, Redis connected lazily; health = `NOT_SERVING` until Redis answers | readiness gate is enough |
| recommendation-service | starts if the address env is set; catalog connected lazily | none |
| frontend, checkout | start regardless (lazy `grpc.NewClient`); requests fail with 500/`INTERNAL` until backends are up | none; consider rolling frontend out last |
| ad / currency / payment / email / shipping | no dependencies | — |
| observability endpoints | all services start fine without Alloy/Pyroscope; exporters just log errors | none |

Suggested install order: namespaces + secrets → `product-catalog-db`, `cart-redis` → leaf services → `product-catalog-service`, `cart-service` → `recommendation`, `checkout` → `frontend`.

### 7.5 Security contexts (implemented and verified in the charts)

| Service | `runAsUser` / `runAsGroup` | root FS | Notes |
|---|---|---|---|
| frontend, ad, currency, payment, email, recommendation | `100` / `101` | **read-only** | `USER` in the images is a name, so `runAsNonRoot` needs the explicit numeric UID. Only `ad-service` needs a writable `/tmp` (an `emptyDir`, exec allowed, for the Pyroscope native library); the others ran with a plain read-only root |
| cart-service | `1654` / `1654` | **read-only** | no writable path needed |
| checkout, product-catalog, shipping | `10001` / `10001` (images run as **root**) | **read-only** | static Go binaries, no file writes |
| product-catalog-db | `999` / `999`, `fsGroup: 999` | **read-only** | `emptyDir` at `/tmp` and `/var/run/postgresql`; `PGDATA` in a sub-directory |
| cart-redis | `999` / `1000`, `fsGroup: 1000` | **read-only** | data on the PVC at `/data` |

Every container: `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`, `seccompProfile: RuntimeDefault`, no ServiceAccount token (`automountServiceAccountToken: false`), `enableServiceLinks: false`.
**Verified:** all 42 pods run in a namespace enforcing the `restricted` Pod Security Standard (`pod-security.kubernetes.io/enforce=restricted`), including after a full rollout restart.
These settings plus digest-pinned images, requests/limits and the standard labels are not just convention: Kyverno enforces them in the cluster (non-root via Pod Security, digest + signature, read-only root filesystem, resources, probes, no API token; §13 and `platform-demo-helm/policies/README.md`).

### 7.6 Resources (requests sized from measured idle usage; limits generous)

| Runtime | Request (CPU / mem) | Limit (CPU / mem) | Services |
|---|---|---|---|
| Go | 50m / 32–48Mi (frontend 100m / 48Mi) | 500m / 128Mi (frontend 192Mi) | frontend, checkout, product-catalog, shipping |
| Node | 100m / 64Mi | 500m / 192Mi | currency, payment |
| Python | 100m / 64Mi | 500m / 192Mi | email, recommendation |
| .NET | 100m / 64Mi | 1 / 256Mi | cart |
| JVM | 100m / 192Mi | 1 / 512Mi | ad (`-XX:MaxRAMPercentage=60 -XX:+ExitOnOutOfMemoryError`) |
| Postgres | 100m / 256Mi | 1 / 512Mi | product-catalog-db |
| Redis | 50m / 64Mi | 500m / 128Mi (+ `--maxmemory 96mb`) | cart-redis |

Measured idle usage in the test stack: Go 10–22 MiB, Node/Python 32–43 MiB, cart 38 MiB, ad 114–131 MiB, Postgres 34–62 MiB, Redis 3–6 MiB; CPU is single-digit millicores for Go, 10–30m for the others. HPA targets are 70 % of the CPU *request*, which is why the requests are not smaller. Re-measure under your real load.

### 7.7 Scaling & gRPC load balancing

All application services are stateless and safe to run with multiple replicas. **But** gRPC uses long-lived HTTP/2 connections, and every client here uses default `pick_first` (Go `grpc.NewClient`, Python `insecure_channel`) against a `ClusterIP` Service. Result: a client pins to **one** backend pod per connection and extra replicas get little or no traffic. Options:

- Keep `replicas: 1` per service for this demo (the earlier chart did).
- Use a service mesh / L7 proxy (Linkerd, Istio, Envoy gateway) for per-request balancing.
- Use a headless Service with `dns:///svc:port` targets **and** `round_robin` service config in each client (code change).

HPA on CPU only helps once load is spread. Add `PodDisruptionBudget` only when `replicas > 1`.

### 7.8 Termination

| Service group | Behaviour on `SIGTERM` | `terminationGracePeriodSeconds` |
|---|---|---|
| Go (frontend, checkout, product-catalog, shipping), .NET, Java | exits promptly (default runtime handling) | default 30 is fine; ad: 10–15 |
| Node (currency, payment) | a handler flushes telemetry but never exits → runs until `SIGKILL` | set **5** |
| Python (email, recommendation) | runs as PID 1 with no handler → signal ignored → runs until `SIGKILL` | set **5** |

(Fixes belong in code: exit in the Node handler; install a `SIGTERM` handler in Python or use `tini`.)

### 7.9 Exposure, TLS and NetworkPolicy

- **Ingress/Gateway → `frontend:80` only.** Use **HTTPS**: the session and currency cookies are `Secure; SameSite=Strict`, so over plain HTTP (non-`localhost`) browsers discard them and carts look empty. `kubectl port-forward` to `localhost` works; `ENABLE_SINGLE_SHARED_SESSION=true` is a demo workaround that shares one cart across all visitors.
- The Go services' debug `/metrics` and (shipping always; checkout/product-catalog when profiling is on) `/debug/pprof` are unauthenticated — the charts do not expose those ports; keep it that way.
- Network policy (implemented in `platform-common`; default deny, then allow). With `networkPolicy.type: cilium` every rule that targets another workload additionally requires **mutual authentication** (SPIFFE/SPIRE) and the Cilium Gateway is admitted as the `ingress` entity; with the default `kubernetes` type it is a plain NetworkPolicy:

| From | To | Port |
|---|---|---|
| ingress (Cilium Gateway / controller) | frontend | 8080 |
| frontend | product-catalog, currency, cart, recommendation, checkout, shipping, ad | 3550, 7000, 7070, 8080, 5050, 50051, 9555 |
| checkout | product-catalog, cart, currency, shipping, payment, email | 3550, 7070, 7000, 50051, 50051, 8080 |
| recommendation | product-catalog | 3550 |
| product-catalog | product-catalog-db | 5432 |
| cart | cart-redis | 6379 |
| all app workloads | Alloy (`monitoring` namespace) | 4317 (OTLP/gRPC); 4040 (profiles, all except cart) |
| all | kube-dns | 53 |

---

## 8. Helm charts

The charts live in `platform-demo-helm/` and are built on a shared library chart so all twelve workloads are hardened and shaped identically. They were installed and exercised on throw-away kind clusters (then deleted); the pull-request pipeline lints, renders, schema-checks and misconfiguration-scans them on every change (§13).

### 8.1 Layout

```
platform-demo-helm/
├── platform-common/        library chart: Deployment, StatefulSet, Service, HPA, PDB, NetworkPolicy, ServiceAccount, Ingress, env/probe/security helpers
├── ad-service/ cart-service/ checkout-service/ currency-service/ email-service/
│   frontend/ payment-service/ product-catalog-service/ recommendation-service/ shipping-service/   (Deployments)
├── product-catalog-db/ cart-redis/                                                                  (StatefulSets)
├── platform-demo/          umbrella chart: installs everything; values-dev.yaml, values-prod.yaml
├── platform-gateway/       Gateway (Cilium GatewayClass) + HTTPRoutes: HTTP→HTTPS redirect, shop, grafana, argocd, hubble
├── monitoring-local/       single-replica Alloy, Mimir, Loki, Tempo, Pyroscope, Grafana for laptops (not for real clusters)
├── monitoring/             vendored upstream charts (unchanged)
├── envs/dev/               what runs in dev: workloads.yaml, addons.yaml, values/<app>.yaml (digests), values/common.yaml
├── gitops/                 Argo CD: values, bootstrap root, AppProjects, ApplicationSets, self-management (README.md)
└── kind/                   up.sh / down.sh, cluster + Cilium values, TLS script and the runbook (README.md)
```

Resource names equal the chart name (`checkout-service`, `cart-redis`, …) **on purpose** — those are the DNS names the services are configured with. Charts are namespace-agnostic: install with `-n <namespace>`; all services are expected in one namespace (short names like `cart-service:7070`), observability in `monitoring`.

### 8.2 What every chart gives you

| Area | Setting (default) |
|---|---|
| Availability | 2 replicas, **HPA** (CPU 70 %, min 2 / max 5; frontend 6), **PodDisruptionBudget** (`maxUnavailable: 1`, `unhealthyPodEvictionPolicy: AlwaysAllow`), RollingUpdate `maxUnavailable: 0`, `minReadySeconds`, `preStop` sleep + tuned `terminationGracePeriodSeconds`, soft **topology spread** (zone + node) |
| Probes | startup + readiness + liveness (gRPC / HTTP / exec / tcp as appropriate; cart liveness deliberately independent of Redis) |
| Resources | requests and limits on every container, including the init container |
| Security | restricted Pod Security profile (§7.5), no SA token, no service links, read-only root FS with explicit `/tmp` |
| Network | **NetworkPolicy** (any CNI) or **CiliumNetworkPolicy + mutual authentication** (`networkPolicy.type: cilium`) per workload: default-deny, only the documented callers; egress only to DNS, the documented dependencies and Alloy |
| Config | env map rendered through `tpl`; DB credentials via `secretKeyRef` and `$(VAR)` expansion into `DATABASE_URL`; `global.monitoring.*` shared |
| Data | StatefulSets with `volumeClaimTemplates`, retention policy `Retain`, headless + ClusterIP Services; Redis AOF on, bounded memory, `noeviction` |
| Hygiene | standard `app.kubernetes.io/*` labels, `values.schema.json` per chart, `NOTES.txt`, image by `digest` > `tag` > `appVersion`, `global.imageRegistry` mirror prefix |
| Ingress | `frontend` only (`ingress.enabled`, class, TLS) — **needs HTTPS** because of the `Secure` cookies |

### 8.3 Install

```bash
cd platform-demo-helm/platform-demo

kubectl create namespace platform-demo
kubectl label ns platform-demo pod-security.kubernetes.io/enforce=restricted

# small cluster / laptop (1 replica, no HPA/PDB; demo DB password):
helm upgrade --install platform-demo . -n platform-demo -f values-dev.yaml

# default HA profile (needs metrics-server for the HPAs):
helm upgrade --install platform-demo . -n platform-demo \
  --set product-catalog-db.auth.existingSecret=product-catalog-db-credentials   # or --set product-catalog-db.auth.password=<alphanumeric>

# one service on its own:
cd ../checkout-service && helm dependency update && helm upgrade --install checkout-service . -n platform-demo
```

`helm dependency update` vendors `platform-common` into each chart's `charts/` (and the charts into the umbrella). Re-run it after editing `platform-common` or a service chart.

### 8.4 The values contract (same shape in every chart)

`image{registry,repository,tag,digest,pullPolicy}` · `replicaCount` · `strategy` · `terminationGracePeriodSeconds` · `preStopSleepSeconds` · `serviceAccount` · `containerPorts` · `service` · `telemetry` · `env` / `secretEnv` / `extraEnv` / `envFrom` · `initContainers` · `startupProbe`/`readinessProbe`/`livenessProbe` · `resources` · `podSecurityContext` / `securityContext` · `tmpVolume` · `volumes`/`volumeMounts` · `persistence` (stores) · `autoscaling` · `podDisruptionBudget` · `topologySpread` · `networkPolicy{ingress,egress}` · `ingress` · `podAnnotations`/`podLabels`/`nodeSelector`/`tolerations`/`affinity`/`priorityClassName` · `global{imageRegistry,monitoring{namespace,otlpEndpoint,pyroscopeAddr}}`.

### 8.5 What was verified (kind, 1 control-plane + 2 workers, k8s 1.32, metrics-server, a stub collector)

- `helm lint` clean on all 13 charts; umbrella renders 51 objects (dev) / 71 (HA).
- Full install in ~45 s, zero restarts; complete browse → cart → checkout through the frontend (all services + Postgres + Redis).
- **NetworkPolicies enforced:** frontend → checkout allowed; frontend → payment and frontend → database blocked; an unlabelled pod cannot reach checkout or frontend.
- Telemetry from the pods reached the collector across the egress rules; apps tolerate a missing collector.
- `restricted` Pod Security enforced on the namespace, then a full rollout restart: all 42 pods re-admitted.
- **HPA:** targets populate; under load frontend scaled 2→6, currency 2→5, cart 2→5 and product-catalog 2→4 (checkout stayed at 2).
- PDBs report the expected allowed disruptions; replicas spread over both workers.
- **2026-10-17 verification of the final setup (single-node kind, Cilium 1.20.2, Argo CD, `kind/up.sh` from scratch):** 16 Applications Synced/Healthy; browse → cart → checkout through the Cilium Gateway over HTTPS returned "Your order is complete!"; all ten services in Mimir and Tempo and in Loki; an intruder pod from another namespace was dropped (`Policy denied`); a pod carrying the `frontend` label but a new identity was dropped with `Authentication required` while SPIRE was down and connected once it was back (mutual authentication is enforced, labels alone are not enough).

### 8.6 Before you deploy

1. **Images:** the ten services are on Docker Hub, signed, and pinned by digest in the charts (§4). Nothing to push. To move to the `v1.0.0` (or a later) release, bump `image.digest` in each chart in one PR; the Helm CI re-verifies the signatures.
2. **Provide the DB credentials** (`auth.existingSecret` or `auth.password`); the install fails fast otherwise.
3. **Install the monitoring stack** so that `alloy.monitoring.svc.cluster.local:4317/4040` resolves (or set `global.monitoring.*`). Apps start fine without it.
4. **HTTPS for the frontend** (ingress with TLS) — otherwise carts appear empty.
5. Make sure the cluster **enforces network policy** (Cilium here; on AWS decide between Cilium and the VPC CNI's NetworkPolicy support - the charts support both via `networkPolicy.type`) and has **metrics-server** for the HPAs (the kind dev profile disables HPAs).
6. `cart-redis` uses a digest-pinned upstream image (`redis:7-alpine`); re-pin deliberately. `product-catalog-db` is the temporary local-testing image; on AWS use RDS instead (point `product-catalog-service` at it with `database.host` and the credentials Secret).

### 8.7 Caveats worth knowing

- **gRPC and HPA:** a `ClusterIP` balances per connection and every client uses `pick_first`, so scaled-out replicas receive little traffic from long-lived connections. HPA still reacts to CPU and new connections spread, but real balancing needs a mesh/L7 proxy or headless services with client-side `round_robin` (code change).
- The soft topology spread keeps single-node clusters working; `values-prod.yaml` shows the hard per-node variant (`DoNotSchedule`).
- Single-replica StatefulSets have no PDB (it would only block node drains). They are not an HA database.
- The charts do not create the `monitoring` namespace or the Alloy `Service`; the stub used for verification was a one-off.

### 8.8 Local cluster (kind + Cilium + Argo CD)

`platform-demo-helm/kind/up.sh` builds everything from scratch (~10-15 min), `kind/down.sh` removes it; `platform-demo-helm/kind/README.md` is the runbook and holds the gotchas, `platform-demo-helm/gitops/README.md` explains how Argo CD deploys the repository.

- **Cluster:** kind `platform-local`, one node, **no default CNI and no kube-proxy**. **Cilium 1.20.2** is the CNI, replaces kube-proxy (eBPF), serves the **Gateway API** (host-network mode: listeners 8080/8443 published as 80/443; GatewayClass `cilium`), runs Hubble, and provides **mutual authentication** through the bundled SPIRE (persistent CA).
- **Argo CD 3.5** (chart 10.10.0) is installed once by `up.sh`; its root Application then creates the projects, two ApplicationSets (workloads, add-ons) and itself. 16 Applications: 12 workloads, `dev-platform-gateway`, `dev-monitoring-local`, `argocd`, `platform-root`.
- **Hostnames** (Windows hosts file and WSL `/etc/hosts`): `127.0.0.1 shop.platform-demo.test grafana.platform-demo.test argocd.platform-demo.test hubble.platform-demo.test`. HTTPS is required because the frontend's cookies are `Secure`; trust `kind/certs/ca.crt` to drop the warning.
- **Bootstrapped outside Git** (by `up.sh`): Gateway API CRDs, Cilium, the TLS Secret, the `platform-demo` namespace and the random DB credentials Secret, Argo CD itself.
- **Footprint:** ~4.8 GB of 7.5 GB used with everything running (1 replica per service).
- **Gotchas that cost hours** (details in `kind/README.md`): the SPIRE image/chart incompatibility (run the server as root), new identities needing 10-15 s, Argo CD needing server-side diff for Gateway API routes, HPAs without metrics-server marking apps Degraded, musl + DNS search domains (`ndots: 2`), `kind load docker-image` not working here.

---

## 9. Cross-cutting findings & gotchas

Ordered by how likely they are to bite. Items marked ✅ were fixed during the OTLP/Helm work.

1. **Frontend cookies are `Secure`** → carts break over plain HTTP (non-localhost). Needs TLS at ingress. (§7.9)
2. ✅ **`product-catalog-service` exits if Postgres isn't up yet** → the chart now has a `wait-for-db` init container.
3. **Service names/ports are baked into defaults and compose values** — Kubernetes Service names must equal the chart names (they do).
4. ✅ **Two OTLP flavours (gRPC vs HTTP) and different env var names** → unified on OTLP/gRPC + `OTEL_EXPORTER_OTLP_ENDPOINT` (§6).
5. ✅ **Dashboards required `job`/`service_name` = service name and metric names** → preserved/updated (§6.3); the compose Alloy config was replaced by `config.otlp.alloy`.
6. **gRPC doesn't load-balance across replicas** via `ClusterIP`. (§8.7)
7. **`SIGTERM` is ignored** by Node (handler never exits) and Python (PID 1) → charts set `terminationGracePeriodSeconds: 5` for those.
8. **Three Go images run as root and are `amd64`-only** (checkout, product-catalog, shipping); the charts override the UID. On arm64 nodes they fail with `exec format error`.
9. **Health checks are static** (except cart's Redis ping) — readiness doesn't reflect dependency health. Cart's liveness must not use the Redis-aware check (handled in the chart).
10. **Docker `HEALTHCHECK`s are ignored** by Kubernetes; the Node ones also break if `PORT` isn't set.
11. **`payment-service` logs full card number/CVV/expiry** at `info`, and `email-service` logs recipient addresses; logs now also travel over OTLP into Loki. Fine for test cards; redact before anything real.
12. **`checkout-service` has no rollback**: a shipping failure after a successful charge leaves the customer charged; not idempotent.
13. **Image hygiene:** `product-catalog-db:latest`, and every Grafana-stack image in compose except Tempo is `latest`; Python images install from unpinned `requirements.in`.
14. ✅ **Node idle CPU bug:** currency and payment spun a `setTimeout(0)` loop to measure event-loop lag and sat at ~25 % CPU while idle — that would have kept an HPA scaling out. Replaced with `monitorEventLoopDelay` (≈1 %).
15. ✅ **`cart-service` build was broken by two things:** the ineffective `.dockerignore` (host `src/obj` leaked into the image build) and new advisories on OpenTelemetry .NET 1.15.0 (`TreatWarningsAsErrors` + NuGet audit). Fixed with a root `.dockerignore` and 1.15.3 packages.
16. **`cart-service` data model:** no TTL, `EmptyCart` overwrites, `AddItem` isn't atomic. Redis has no auth (NetworkPolicy isolates it).
17. **`product-catalog` cache is per pod**; refresh with `SIGUSR1` (`kubectl exec … -- kill -USR1 1`) or restart.
18. **pprof exposure:** shipping always serves `/debug/pprof` on its (unexposed) debug port; checkout/product-catalog when `ENABLE_PROFILING=1`. The charts do not expose these ports.
19. **`shipping.ShipOrder` crashes the process on a nil address** (no recovery interceptor) — latent, only `checkout` calls it with a validated address.
20. **Docs drift fixed in the READMEs:** `docs/` folders are empty; `frontend` entrypoint is `./cmd/server`; `ENV_PLATFORM` isn't read; `checkout` does not require downstreams at boot; `ad-service` returns *all* ads per matched category; cart logs aren't JSON by default.
21. **`shopping-assistant-service`** is GCP-only, unwired, no health endpoint/telemetry — excluded from the charts.
22. **Pyroscope Java agent needs an executable `/tmp`**; a `noexec` tmpfs makes `ad-service` crash at startup (an `emptyDir` is fine).

## 10. Local stack (docker compose)

`platform-demo-observability/docker-compose.yaml` runs the whole thing plus the LGTM stack on one bridge network (`observability`), with service-name DNS.

```bash
cd platform-demo-observability

# OTLP/gRPC stack (uses the v1.0.0 images from Docker Hub, :1.0.0):
docker compose -f docker-compose.yaml -f docker-compose.otlp.yaml up -d

# original stack (pre-OTLP digests, Prometheus scraping, OTLP/HTTP):
# docker compose up -d

docker compose ps
open http://localhost        # frontend (host 80 → 8080)
open http://localhost:3000   # Grafana (anonymous admin)
```

`docker-compose.otlp.yaml` is an overlay: it swaps Alloy's config for `alloy/config.otlp.alloy`, points each app at `http://alloy:4317` / `:4040`, uses the `1.0.0` images and publishes only the main service ports. It was smoke-tested end to end (all ten `job`s in Mimir, all services in Loki, Grafana datasources provisioned).

Host port map (OTLP overlay; the original file additionally publishes the old metrics ports 9090–9098):

| Component | Host → container | | Component | Host → container |
|---|---|---|---|---|
| frontend | 80→8080 | | ad-service | 9555→9555 |
| product-catalog-service | 3550→3550 | | cart-service | 7070→7070 |
| checkout-service | 5050→5050 | | currency-service | 7000→7000 |
| shipping-service | 50051→50051 | | payment-service | 50052→50051 |
| email-service | 8080→8080 | | recommendation-service | 8081→8080 |
| product-catalog-db | 5432→5432 | | cart-redis | 6379→6379 |
| Grafana | 3000 | | Alloy | 12345 UI, 4317 OTLP/gRPC (original file also 4318 HTTP) |
| Mimir | 9009 | | Loki | 3100 |
| Tempo | 3200, 4319→4317 | | Pyroscope | 4040 |

Environment per service in that file is the reference for the minimum viable config of each container (§5.2, §6.1). Volumes: `grafana_data`, `mimir_data`, `loki_data`, `tempo_data`, `pyroscope_data`, `postgres_data`, `redis_data`.

---

## 11. Repository map

### Workspace layout (local checkout)

Every directory below `apps/`, `deploy/`, `platform/` is its own Git repository (cloned side by side); `docs/` is the repository `platform-demo-docs` (this file). Grouping is by *function*, so new work has an obvious home:

```
platform-demo/
  apps/        what we build: deployable services, one repo each (11 + a dormant GCP-only assistant without a repo);
               local folder = repo name without the `app-` prefix (apps/frontend is github.com/platform-boutique/app-frontend)
  deploy/      what runs where: platform-demo-helm (charts, envs/<env>/, gitops/, policies/, kind/)
  platform/    the machinery around the apps: shared CI/CD workflows, observability configs
               platform-demo-backstage (the developer portal) and its two plugins
               (platform-demo-backstage-plugin-hubble, -releases); future: synthetic tests, monitors/alerts, AI tooling
  infra/       cloud and cluster infrastructure (future: terraform/ for the platform cluster, crossplane/)
  docs/        platform-demo-docs: this master reference, the handoff, history
  platform-demo.code-workspace    multi-root VS Code workspace for all of the above
```

Convention for what is added later: a new service -> `apps/<name>` (+ a line in `deploy/.../envs/<env>/workloads.yaml`); anything that provisions clouds or clusters (Terraform, Crossplane compositions) -> `infra/<name>`; developer portal, test/monitoring/alerting code, AI tooling that serves the platform -> `platform/<name>`; environment state -> only `deploy/platform-demo-helm/envs/`. **Naming on GitHub** (an organisation has no folders, so the category is in the name): `app-<service>` for services, `infra-<name>` for infrastructure code (planned: `infra-terraform`, `infra-crossplane`), `platform-demo-<name>` for the platform itself (deploy, platform tooling, docs). Every repository also has the custom property `category` (`app`, `deploy`, `platform`, `infra`, `docs`; filter the org list with `props.category:app`) and topics. The organisation front page (repo `platform-boutique/.github`, `profile/README.md`) lists them by category. Roadmap: the public Project board https://github.com/orgs/platform-boutique/projects/1 (field `Area`).

### Repositories (all in the GitHub organisation `platform-boutique`; old `mladenovskistefan111/<repo>` URLs redirect)

| Group | Repository | README | Contents |
|---|---|---|---|
| apps | [`app-frontend`](https://github.com/platform-boutique/app-frontend) | [README](https://github.com/platform-boutique/app-frontend#readme) | HTTP storefront (Go) |
| apps | [`app-ad-service`](https://github.com/platform-boutique/app-ad-service) | [README](https://github.com/platform-boutique/app-ad-service#readme) | ads (Java) |
| apps | [`app-cart-service`](https://github.com/platform-boutique/app-cart-service) | [README](https://github.com/platform-boutique/app-cart-service#readme) | carts on Redis (.NET) |
| apps | [`app-checkout-service`](https://github.com/platform-boutique/app-checkout-service) | [README](https://github.com/platform-boutique/app-checkout-service#readme) | order orchestration (Go) |
| apps | [`app-currency-service`](https://github.com/platform-boutique/app-currency-service) | [README](https://github.com/platform-boutique/app-currency-service#readme) | currency conversion (Node/TS) |
| apps | [`app-email-service`](https://github.com/platform-boutique/app-email-service) | [README](https://github.com/platform-boutique/app-email-service#readme) | email stub (Python) |
| apps | [`app-payment-service`](https://github.com/platform-boutique/app-payment-service) | [README](https://github.com/platform-boutique/app-payment-service#readme) | payment simulation (Node/TS) |
| apps | [`app-product-catalog-service`](https://github.com/platform-boutique/app-product-catalog-service) | [README](https://github.com/platform-boutique/app-product-catalog-service#readme) | catalog API on Postgres (Go) |
| apps | [`app-product-catalog-db`](https://github.com/platform-boutique/app-product-catalog-db) | [README](https://github.com/platform-boutique/app-product-catalog-db#readme) | Postgres image with schema + seed (temporary, local testing; RDS on AWS) |
| apps | [`app-recommendation-service`](https://github.com/platform-boutique/app-recommendation-service) | [README](https://github.com/platform-boutique/app-recommendation-service#readme) | recommendations (Python) |
| apps | [`app-shipping-service`](https://github.com/platform-boutique/app-shipping-service) | [README](https://github.com/platform-boutique/app-shipping-service#readme) | quotes + tracking IDs (Go) |
| apps | `shopping-assistant-service` (local only, no repo) | local README | Gemini/AlloyDB assistant (dormant, GCP-only) |
| deploy | [`platform-demo-helm`](https://github.com/platform-boutique/platform-demo-helm) | [README](https://github.com/platform-boutique/platform-demo-helm#readme) | Helm charts (library + 12 workloads + umbrella + Gateway + local monitoring), `envs/`, `gitops/` (Argo CD), `policies/` (Kyverno), `kind/` runbook |
| platform | [`platform-demo-workflows`](https://github.com/platform-boutique/platform-demo-workflows) | [README](https://github.com/platform-boutique/platform-demo-workflows#readme) | reusable GitHub Actions workflows (Go, Node, Python, Java, .NET CI; Docker build/scan; Docker promote/sign; secret scan; workflow lint) and the generators/protection scripts in `tools/` |
| platform | [`platform-demo-observability`](https://github.com/platform-boutique/platform-demo-observability) | — | compose stack: Alloy, Mimir, Loki, Tempo, Pyroscope, Grafana + dashboards; `alloy/config.otlp.alloy` + `docker-compose.otlp.yaml` for the OTLP setup |
| docs | [`platform-demo-docs`](https://github.com/platform-boutique/platform-demo-docs) | this file | master reference, `HANDOFF.md`, `OTLP-MIGRATION-PLAN.md`, `history/` |

Each `*-service` repo has thin `.github/workflows/ci.yaml` and `release.yaml` callers of the shared workflows (§13) and a `tests/` (or `__tests__/`) folder. `tests/e2e` and `tests/integration` of cart, frontend and product-catalog target the removed kind-based job and are not run by CI.

All gRPC APIs share protobuf package `hipstershop` (the Online Boutique heritage). Services keep their own copy of the `.proto` files they use (`proto/` or `src/protos/`), including client-side definitions of the services they call.

---

## 12. Next steps

Priority order; details, decisions and context are in [`HANDOFF.md`](HANDOFF.md). Kyverno (admission control, §13) is done.

1. **Make the developer portal real.** Backstage (catalog from `catalog-info.yaml` per repo, TechDocs, Kubernetes + Argo CD + Kyverno + Grafana plugins, own Hubble and Releases plugins) and single sign-on through **Dex** are built and were tested on the local cluster, but exist only on local branches (nothing pushed). Remaining: create the three new repositories and push, the GitHub OAuth App and a `maintainers` team, catalog discovery from GitHub, the portal's first release, TechDocs publishing in CI; see [`HANDOFF.md`](HANDOFF.md) §0 and the portal's own documentation (`platform-demo-backstage/docs`, rendered in the portal).
2. Infrastructure as code for the platform cluster: **Terraform** (cluster, networking, state) and **Crossplane** (self-service infrastructure), under `infra/`; on AWS: RDS instead of `product-catalog-db`, cert-manager + ingress/TLS, External Secrets, SPIRE storage on EBS, Cilium installed by the cluster pipeline, a registry without anonymous pull limits (ECR pull-through) so image-signature verification can be enforced.
3. Observability depth: synthetic tests, monitors and alerts (SLOs per service, alert rules in Mimir/Grafana), Hubble metrics into Mimir (to retire the Hubble UI/relay pods), policy-report metrics from Kyverno.
4. Semantic-versioning automation and enforcement (§13): conventional-commit PR titles as a required check, release-please opening release PRs, strict `vX.Y.Z` tag filter; automated digest-promotion PRs and a second environment (`envs/prod`).
5. AI tooling on top of the platform.
6. Adopt Cilium (and the Gateway API CRDs) into GitOps; network policies for the `argocd` and `monitoring` namespaces; WireGuard encryption if data-path encryption is required (mutual auth only authenticates).
7. Hardening backlog: redact card data in payment (it logs full card data), graceful `SIGTERM` in Node/Python, JSON logs for cart, headless Services or a mesh for real gRPC balancing, Dockerfile base images pinned by digest, review the Node `.trivyignore` entries; Argo CD SSO instead of the local admin and rotating the initial admin secret.

To continue in a new chat, paste this file, `HANDOFF.md` and the README of what you are working on.

---

## 13. CI/CD, branching and releases

**Model: trunk-based.** `main` is the trunk; work happens on short-lived `feat/*`, `fix/*`, `docs/*`, `chore/*` branches merged by **squash PR**. There is no production branch: a release is an annotated semver tag `vX.Y.Z` on `main`, and environments are values files in `platform-demo-helm` (promotion = a PR that bumps image digests).

**Pipelines never run on push.** `ci.yaml` runs on `pull_request` (and manual dispatch); `release.yaml` runs on a `v*` tag or manual dispatch (`publish=true`). Both are thin callers of [`platform-demo-workflows`](https://github.com/platform-boutique/platform-demo-workflows#readme) (currently **v2.0.2**), pinned by commit SHA (`# v2.0.1` comment), with explicit per-job `permissions:`.

| Stage | What happens |
|---|---|
| language gates | format/lint, gitleaks, build, unit tests (+ E2E and gRPC smoke where enabled), SCA (govulncheck / npm audit / pip-audit / Trivy fs / `dotnet list --vulnerable`), SAST (gosec / Semgrep / bandit) |
| image | build once, stage on ghcr.io, Trivy-scan that exact digest (HIGH/CRITICAL fail) |
| release (tag only) | refuse commits not on `main`; copy the scanned digest to Docker Hub (no rebuild); tags `:X.Y.Z`, `:sha-<short>`, `:latest`; cosign 3.1.3 keyless sign (classic `.sig` format - Docker Hub has no OCI referrers API) + self-verify; SBOM attached as a signed `spdxjson` attestation + self-verify |

**Helm repo pipeline (PR only):** `helm lint` on every chart, render of the umbrella (dev/default/prod) + gateway + monitoring-local **+ every app of every environment exactly as Argo CD renders it** (`.github/scripts/render-envs.py`), kubeconform also over the Argo CD manifests, kubeconform (strict, CRD schemas), Trivy config scan (app and gateway charts must be clean; `monitoring-local` report-only), cosign gate (every deployed `mladenovskistefan/*` image pinned by digest and signed; `product-catalog-db` allow-listed as temporary), secret scan, workflow lint, and the **Kyverno policy tests** (unit tests for every policy; every rendered workload/add-on of every environment is evaluated against the policies).

**Guardrails:** branch protection on `main` in all 14 repos (PR required, required checks, linear history, no force-push or deletion, enforced for admins; 0 approvals because there is one maintainer), immutable `v*` tags (ruleset), GitHub secret scanning + push protection, CODEOWNERS, Dependabot (weekly, 7-day cooldown, minor/patch grouped, each major its own PR), and local `pre-commit` hooks (gitleaks on the staged diff at commit time, on the pushed commits at push time; `--no-verify` bypasses them, push protection and CI do not).

Set up the hooks in a fresh clone with `pipx install pre-commit && pre-commit install`.

**Dependency policy:** every action is pinned to the newest release that is at least a week old (supply-chain cooldown). `gradle/actions` stays on **v5** because v6 puts its caching component under proprietary terms that were not accepted on anyone's behalf. Semgrep runs from a pinned `pip install` (the `semgrep-action` has been unmaintained since 2020). Dependabot majors for toolchains (node, typescript, eslint, python base images, gradle wrapper, junit, ...) were closed with "ignore this major version"; take them as deliberate upgrades.

### Continuous delivery (Argo CD)

Delivery is pull-based: the pipelines above end at a signed image on Docker Hub; **nothing pushes to the cluster**. `platform-demo-helm` is the desired state (`envs/<env>/` + charts); Argo CD (`gitops/`) renders it and reconciles the cluster. Two ApplicationSets (workloads, add-ons) generate one Application per environment and app from `envs/*/workloads.yaml` / `addons.yaml`; Argo CD manages its own installation too. **Promotion = a PR** changing `image.digest` in `envs/<env>/values/<app>.yaml`; the Helm CI renders it, validates it and verifies the digest is signed before it can merge. See `platform-demo-helm/gitops/README.md`.

### Network security (Cilium)

The kind cluster runs Cilium as CNI (kube-proxy replacement, Gateway API, Hubble) with SPIFFE/SPIRE **mutual authentication**. Workload policies are `CiliumNetworkPolicy` objects generated by `platform-common` (`networkPolicy.type: cilium`): default-deny, explicit callers/dependencies, and `authentication: mode: required` on every workload-to-workload rule. Proven on 2026-10-17: an intruder in another namespace is dropped (`Policy denied`); a pod with an allowed label but no valid identity is dropped (`Authentication required`). Limits: Beta feature; authenticates but does not encrypt; identity is label-derived, so who may set labels is controlled by Kyverno (below).

### Admission control (Kyverno)

Kyverno 1.19 is deployed by Argo CD (`gitops/apps/application-kyverno.yaml`, project `security`) with the policies from `platform-demo-helm/policies/` (CEL policy types, per-environment values). **Enforced for `platform-demo`**, report-only for `monitoring` and `argocd`. Full catalog, rollout procedure, exceptions and break-glass: [`policies/README.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/policies/README.md).

| Concern | What is enforced |
|---|---|
| Network identity | pods in `platform-demo` may only carry one of the 12 workload names in `app.kubernetes.io/name`, must run under the ServiceAccount of the same name, and cannot change the label (Cilium policy and mutual auth identify pods by that label) |
| Supply chain | images only from allowed repositories, pinned by digest, never `:latest`; images of ours verified against the keyless cosign signature (same identity regexp as the Helm CI gate) - in dev this one policy **audits** until a read-only Docker Hub token exists (anonymous pull limit) |
| Pod hardening | read-only root filesystem, requests + memory limit, readiness/liveness probes, dedicated ServiceAccount, no API token; mutations add safe defaults (token off, seccomp RuntimeDefault, `backstage.io/kubernetes-id`) |
| Exposure | Services are ClusterIP only, HTTPRoutes only for `*.platform-demo.test`, only Argo CD / admins change network policy, Gateways and routes |
| Operations | `kubectl exec/attach/debug` into workloads only with the break-glass annotation; namespaces keep their Pod Security level; a generated baseline (quota, LimitRange, default-deny CiliumNetworkPolicy) for workload namespaces |

Tested three ways: `kyverno test` (49 cases), the CI evaluation of the rendered environments, and a live suite (`policies/tests/live/adversarial.sh`, 29 cases that each assert which policy denies). Gotchas worth knowing before touching a policy (absent map keys are errors in CEL, `matchImageReferences` globs match the string as written in the pod, the Docker Hub rate limit) are in the policies README.
