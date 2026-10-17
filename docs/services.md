# Service inventory

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
