# Helm charts

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
