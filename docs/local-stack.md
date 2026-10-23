# Local stack (docker compose)

The only way to run the shop on a laptop is docker compose; there is no local Kubernetes environment. The cluster is AWS EKS ([Architecture](architecture.md)), and the observability stack (Alloy, Mimir, Loki, Tempo, Pyroscope, Grafana) exists **only here** for now. The files live in the [`platform-demo-observability`](https://github.com/platform-boutique/platform-demo-observability) repository, whose own docs ([running the stack](https://github.com/platform-boutique/platform-demo-observability/blob/main/docs/running.md), dashboards) are the owner of the details.

`platform-demo-observability/docker-compose.yaml` runs the whole thing plus the LGTM stack on one bridge network (`observability`), with service-name DNS.

```bash
cd platform-demo-observability

# OTLP/gRPC stack (uses the v1.0.0 images from Docker Hub, :1.0.0):
docker compose -f docker-compose.yaml -f docker-compose.otlp.yaml up -d

# original stack (pre-OTLP digests, Prometheus scraping, OTLP/HTTP):
# docker compose up -d

docker compose ps
open http://localhost        # frontend (host 80 → 8080; plain HTTP is fine on localhost)
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

Environment per service in that file is the reference for the minimum viable config of each container (§5.2, §6.1). The compose stack has no TLS, no network policy and no admission control: it shows that the services work, not that the platform around them does. Volumes: `grafana_data`, `mimir_data`, `loki_data`, `tempo_data`, `pyroscope_data`, `postgres_data`, `redis_data`.

---
