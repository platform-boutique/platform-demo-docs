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
