# Cross-cutting findings and gotchas

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
