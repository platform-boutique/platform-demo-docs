# Helm charts

The charts live in [`platform-demo-helm`](https://github.com/platform-boutique/platform-demo-helm) and are built on a shared library chart so all twelve workloads are hardened and shaped identically. The same repository holds the GitOps configuration that makes Argo CD deploy them to the EKS cluster (§8.8). The pull-request pipeline lints, renders (exactly as Argo CD will), schema-checks, misconfiguration-scans and policy-tests them on every change (§13). **Nothing here has been verified on EKS yet**; see §8.5.

### 8.1 Layout

```
platform-demo-helm/
├── platform-common/        library chart: Deployment, StatefulSet, Service, HPA, PDB, NetworkPolicy, ServiceAccount, Ingress, env/probe/security helpers
├── ad-service/ cart-service/ checkout-service/ currency-service/ email-service/
│   frontend/ payment-service/ product-catalog-service/ recommendation-service/ shipping-service/   (Deployments)
├── product-catalog-db/ cart-redis/                                                                  (StatefulSets)
├── platform-demo/          umbrella chart: installs everything; values-dev.yaml, values-prod.yaml
├── platform-gateway/       Gateway (Cilium GatewayClass, plain HTTP on 8080 behind the ALB) + HTTPRoutes: shop, argocd, hubble, backstage, dex
├── dex/ backstage/         add-on charts: single sign-on and the developer portal
├── cluster-bootstrap/      namespaces with their Pod Security / Kyverno / gVisor labels, RuntimeClass gvisor, the one-off Cilium restart Job
├── platform-network-policies/  default-deny + allow rules for argocd, sso, backstage, gateway-system
├── external-secrets-config/    ClusterSecretStores + ExternalSecrets (Parameter Store, Secrets Manager)
├── policies/               Kyverno policies (CEL policy types) and their tests
├── monitoring/             vendored upstream charts (unchanged; not deployed to the cluster for now)
├── envs/dev/               what runs in dev: workloads.yaml, addons.yaml, values/<app>.yaml (digests), values/common.yaml, values/cluster-bootstrap.yaml (the namespace list)
└── gitops/                 Argo CD: root Application, AppProjects, one Application per layer, ApplicationSets, self-management (README.md)
```

Resource names equal the chart name (`checkout-service`, `cart-redis`, …) **on purpose** — those are the DNS names the services are configured with. Charts are namespace-agnostic: install with `-n <namespace>`; all services are expected in one namespace (short names like `cart-service:7070`); on EKS that is `platform-demo`.

### 8.2 What every chart gives you

| Area | Setting (default) |
|---|---|
| Availability | chart defaults: 2 replicas, **HPA** (CPU 70 %, min 2 / max 5; frontend 6), **PodDisruptionBudget** (`maxUnavailable: 1`, `unhealthyPodEvictionPolicy: AlwaysAllow`), RollingUpdate `maxUnavailable: 0`, `minReadySeconds`, `preStop` sleep + tuned `terminationGracePeriodSeconds`, soft **topology spread** (zone + node). The EKS dev environment overrides this to the small profile: 1 replica, no HPA, PDB or spread (two Spot nodes) |
| Probes | startup + readiness + liveness (gRPC / HTTP / exec / tcp as appropriate; cart liveness deliberately independent of Redis) |
| Resources | requests and limits on every container, including the init container |
| Security | restricted Pod Security profile (§7.5), no SA token, no service links, read-only root FS with explicit `/tmp` |
| Network | **NetworkPolicy** (any CNI) or **CiliumNetworkPolicy + mutual authentication** (`networkPolicy.type: cilium`) per workload: default-deny, only the documented callers; egress only to DNS, the documented dependencies and Alloy |
| Config | env map rendered through `tpl`; DB credentials via `secretKeyRef` and `$(VAR)` expansion into `DATABASE_URL`; `global.monitoring.*` shared |
| Data | StatefulSets with `volumeClaimTemplates`, retention policy `Retain`, headless + ClusterIP Services; Redis AOF on, bounded memory, `noeviction` |
| Hygiene | standard `app.kubernetes.io/*` labels, `values.schema.json` per chart, `NOTES.txt`, image by `digest` > `tag` > `appVersion`, `global.imageRegistry` mirror prefix |
| Entry | `frontend` only. On EKS it is reached through the Cilium Gateway (an HTTPRoute in `platform-gateway`; the `frontend` chart admits the `ingress` entity), after the ALB has terminated TLS. **Needs HTTPS** because of the `Secure` cookies. The chart's own `ingress.*` values are for other clusters |

### 8.3 Install without Argo CD

On EKS nothing is installed by hand: Argo CD applies the charts (§8.8). For any other cluster with a CNI that enforces NetworkPolicy, the umbrella chart installs the whole shop:

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

### 8.5 What has been verified, and where

| Where | What |
|---|---|
| Every pull request (CI) | `helm lint` of every chart, rendering of every environment app as Argo CD renders it, kubeconform with CRD schemas (Kubernetes, Cilium, Gateway API, Argo CD), Trivy configuration scan, a cosign gate (every deployed image pinned by digest and signed by one of our pipelines), Kyverno unit tests plus an evaluation of every rendered workload against the policies, secret scan, workflow lint |
| Earlier, on throw-away local kind clusters (no longer part of the project) | Full install, browse → cart → checkout through the frontend; NetworkPolicy enforcement (an unlabelled or foreign pod is dropped); `restricted` Pod Security with a full rollout restart; HPA scaling under load; Cilium mutual authentication (a pod with the right label but no valid SPIFFE identity is dropped with `Authentication required`) |
| **EKS** | **Not verified.** The platform is merged but not deployed. The kind results are design evidence, not proof for EKS: gVisor, WireGuard, SPIRE start order, the ALB path, External Secrets and Kyverno on the host network all differ. The checklist to run after the first deploy is "After the first real deploy: verify" in the Helm repository's [`gitops/README.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/gitops/README.md), and the first-deploy risks are in the infra repository's [Architecture](https://github.com/platform-boutique/infra-terraform/blob/main/docs/architecture.md) |

### 8.6 Before you deploy

1. **Images:** the ten services are on Docker Hub, signed, and pinned by digest in `envs/dev/values/<app>.yaml` (§4). To move to a newer release, change `image.digest` in one PR; the Helm CI re-verifies the signatures.
2. **DB credentials:** the shop reads `product-catalog-db-credentials` (`auth.existingSecret`); the install fails fast without a Secret or a password. On EKS the Secret is meant to be created by External Secrets from Parameter Store (untested; confirm at the first deploy). Secrets are never in Git: the table of parameters is in the infra repository's [Secrets](https://github.com/platform-boutique/infra-terraform/blob/main/docs/secrets.md).
3. **Monitoring:** there is none on the EKS cluster for now, so `alloy.monitoring.svc.cluster.local:4317/4040` does not resolve. Apps start fine without it and log export errors; set `global.monitoring.*` once a collector exists.
4. **HTTPS for the frontend:** provided by the ALB on EKS. Anywhere else, ingress with TLS, otherwise carts appear empty.
5. **Network policy and HPA:** Cilium enforces network policy (and mutual authentication) on EKS. The dev profile disables the HPAs; anywhere else they need metrics-server.
6. **Databases:** `cart-redis` uses a digest-pinned upstream image (`redis:7-alpine`); re-pin deliberately. `product-catalog-db` stays a StatefulSet on a `gp3` volume by decision (the shop is expected to move to its own cluster); only Backstage uses RDS.

### 8.7 Caveats worth knowing

- **gRPC and HPA:** a `ClusterIP` balances per connection and every client uses `pick_first`, so scaled-out replicas receive little traffic from long-lived connections. HPA still reacts to CPU and new connections spread, but real balancing needs a mesh/L7 proxy or headless services with client-side `round_robin` (code change).
- The soft topology spread keeps single-node clusters working; `values-prod.yaml` shows the hard per-node variant (`DoNotSchedule`).
- Single-replica StatefulSets have no PDB (it would only block node drains). They are not an HA database, and their EBS volume ties them to one zone.
- The charts do not create a `monitoring` namespace or an Alloy `Service`.

### 8.8 The EKS platform: Argo CD layers

The infrastructure side (VPC, EKS 1.36, Cilium with SPIRE mutual authentication and WireGuard, Spot nodes with gVisor, the default `gp3` StorageClass, RDS for Backstage, the ALB, DNS) is [infra-terraform](https://github.com/platform-boutique/infra-terraform). Terraform installs **only Argo CD** and one root Application; from then on Git is the source of truth. Nothing is applied by hand. The Helm repository's [`gitops/README.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/gitops/README.md) is the owner of the details (waves, projects, first boot, verification checklist, gVisor risks); this page only gives the shape.

```
platform-root (Terraform; path gitops/apps)
 |-- AppProjects (wave -10)
 |-- layer-bootstrap  (wave 0)   namespaces + labels, RuntimeClass gvisor, one-off Job: restart Cilium after SPIRE
 |-- layer-security   (wave 10)  External Secrets -> its config + Reloader -> Kyverno -> policies
 |-- layer-platform   (wave 20)  network policies -> add-ons (Gateway, Dex, Backstage) -> Argo CD managing itself
 `-- layer-workloads  (wave 20)  the shop: 10 services + product-catalog-db and cart-redis (ApplicationSet)
```

- **Why layers.** Sync waves order Applications inside one parent; a wave starts when the previous one is Healthy. So namespaces and the RuntimeClass exist before anything uses them, the secrets and admission control exist before the platform and the shop, and Argo CD applies its own final configuration (local admin off, Dex sign-in) last. ApplicationSet-generated Applications cannot be ordered against each other and rely on retries.
- **Add a workload or add-on:** one line in `envs/dev/workloads.yaml` or `addons.yaml`, values in `envs/dev/values/<app>.yaml`, and its namespace declared in `envs/dev/values/cluster-bootstrap.yaml` (CI fails the pull request otherwise). **Promote an image:** a pull request that changes `image.digest`; Argo CD syncs within about two minutes.
- **Projects** give each layer least privilege (the shop's project may only touch namespaced resources in `platform-demo`).
- **Secrets** reach the cluster through External Secrets (AWS Parameter Store and Secrets Manager); Reloader restarts a workload when its secret changes. See the Helm repository's [`docs/secrets.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/docs/secrets.md).
- **Admission:** Kyverno 1.19, policies catalogued in [`policies/README.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/policies/README.md): `Deny` for the shop's namespace (identity, image source and cosign signature, read-only root filesystem, resources, probes, no API token, break-glass for `kubectl exec`), report-only for the platform namespaces. The admission controller runs on the host network (port 9443) because the EKS control plane cannot reach pods on the overlay.
- **Entry:** the Gateway chart declares one HTTP listener on 8080 and HTTPRoutes for `shop`, `argocd`, `backstage`, `dex` and `hubble` under `platform-demo.click`. Grafana is a disabled entry: no monitoring stack runs on the cluster for now.
- **Hostnames** are DNS records Terraform creates; sign-in to Argo CD and Backstage is GitHub through Dex (the OAuth App's callback is `https://dex.platform-demo.click/callback`).
- **Removed:** the local kind cluster, its scripts and `monitoring-local` are gone; there is no local Kubernetes environment any more. The compose stack ([local stack](local-stack.md)) is the way to run the services on a laptop.

More: the Helm repository's [`docs/`](https://github.com/platform-boutique/platform-demo-helm/blob/main/docs/gitops.md) (GitOps, policies, secrets, SSO and the portal, promotion) and the infra repository's [Pipeline](https://github.com/platform-boutique/infra-terraform/blob/main/docs/pipeline.md) and [Operations](https://github.com/platform-boutique/infra-terraform/blob/main/docs/operations.md).

---
