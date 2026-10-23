# System at a glance

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
