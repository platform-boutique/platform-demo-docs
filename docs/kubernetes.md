# Kubernetes translation guide

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
