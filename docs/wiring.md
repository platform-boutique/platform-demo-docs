# Service wiring

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

Address variables are **`host:port` with no scheme**. All hostnames below are the compose service names — keep the Kubernetes `Service` names identical so these values keep working inside one namespace, or use FQDNs (`<svc>.<ns>.svc.cluster.local`) across namespaces. On the cluster the whole shop lives in one namespace, `platform-demo`, so the short names work unchanged; the charts set these variables from their values files.

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

The telemetry variables are separate (contract in [Observability](observability.md)). The charts default them to `alloy.monitoring.svc.cluster.local:4317` (OTLP/gRPC) and `:4040` (profiles). On the EKS cluster no monitoring stack runs for now, so that name does not resolve: the exporters log errors and the services keep working. Pointing `global.monitoring.*` at a real collector (or switching export off) is an open decision.

### 5.3 Secrets & sensitive values

| Value | Where used | In compose | In Kubernetes (EKS) |
|---|---|---|---|
| Postgres `POSTGRES_USER/PASSWORD/DB` | `product-catalog-db` | `catalog` / `catalog` / `productcatalog` (plain in compose) | `Secret` `product-catalog-db-credentials` (`envFrom`), keys `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`; the password is intended to come from Parameter Store through External Secrets, never from Git. *Untested: confirm the ExternalSecret exists at the first deploy* |
| `DATABASE_URL` (embeds the password) | `product-catalog-service` | plain env | built by the chart from the same Secret (`secretKeyRef` and `$(VAR)` expansion) |
| Redis password (if added) | `cart-service` `REDIS_ADDR` (`host:port,password=…`) | none (no auth) | none today; would be a `Secret` |
| Credit-card data in transit | `frontend` → `checkout` → `payment` | test cards only; **payment logs the full request** | redact in code before real use. Network policy plus mutual authentication and WireGuard protect the path, not the log |
| Backstage database password | the portal | not applicable | generated and rotated (every 7 days) by RDS in Secrets Manager, read by External Secrets; Reloader restarts Backstage when it changes. It never reaches Git or Terraform state |
| Sign-in secrets (GitHub OAuth App, Dex client secrets, session secret), Docker Hub read token (Kyverno) | Dex, Argo CD, Backstage, Kyverno | not applicable | Parameter Store `SecureString` parameters under `/platform-demo/`, typed once by a person or generated; table in the infra repository's [Secrets](https://github.com/platform-boutique/infra-terraform/blob/main/docs/secrets.md) and the Helm repository's [`docs/secrets.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/docs/secrets.md) |
| Google credentials / API key | `shopping-assistant-service` | not deployed | not deployed |

The shop has no other credentials. The services' own gRPC is unauthenticated and unencrypted. In compose that is the whole story; on the cluster the network adds SPIFFE mutual authentication between workloads and WireGuard between nodes (designed and merged, **not yet verified on a live cluster**).

---
