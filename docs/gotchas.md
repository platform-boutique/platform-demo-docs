# Cross-cutting findings and gotchas

Two lists: how the **application** behaves (the things that bite when you deploy or change a service), then what was learned about the **platform** (admission control, GitOps, networking, supply chain, portal). Items marked ✅ were fixed. Policy-specific details are also in `policies/README.md` of `platform-demo-helm`.

## The application

1. **Frontend cookies are `Secure`** → carts break over plain HTTP (non-localhost). TLS ends at the ALB; the Gateway runs plain HTTP behind it, and Cilium's Envoy rewrites `X-Forwarded-Proto` to the scheme of its own connection, so every route sets `X-Forwarded-Proto: https` explicitly. ([Kubernetes translation guide](kubernetes.md))
2. ✅ **`product-catalog-service` exits if Postgres isn't up yet** → the chart has a `wait-for-db` init container.
3. **Service names/ports are baked into defaults and compose values** — Kubernetes Service names must equal the chart names (they do).
4. ✅ **Two OTLP flavours (gRPC vs HTTP) and different env var names** → unified on OTLP/gRPC + `OTEL_EXPORTER_OTLP_ENDPOINT` ([Observability contract](observability.md)).
5. ✅ **Dashboards required `job`/`service_name` = service name and metric names** → preserved/updated; the compose Alloy config was replaced by `config.otlp.alloy`.
6. **gRPC doesn't load-balance across replicas** via `ClusterIP`: a client keeps one HTTP/2 connection. The dev profile runs one replica each, so it does not show.
7. **`SIGTERM` is ignored** by Node (handler never exits) and Python (PID 1) → charts set `terminationGracePeriodSeconds: 5` for those.
8. **Three Go images run as root and are `amd64`-only** (checkout, product-catalog, shipping); the charts override the UID. On arm64 nodes they fail with `exec format error` (one more reason the node pool stays x86).
9. **Health checks are static** (except cart's Redis ping) — readiness doesn't reflect dependency health. Cart's liveness must not use the Redis-aware check (handled in the chart).
10. **Docker `HEALTHCHECK`s are ignored** by Kubernetes; the Node ones also break if `PORT` isn't set.
11. **`payment-service` logs full card number/CVV/expiry** at `info`, and `email-service` logs recipient addresses; logs also travel over OTLP. Fine for test cards; redact before anything real.
12. **`checkout-service` has no rollback**: a shipping failure after a successful charge leaves the customer charged; not idempotent.
13. **Image hygiene:** `product-catalog-db` is a hand-built, unsigned image (allow-listed by name); Python images install from unpinned `requirements.in`; base images are pinned by minor tag, not digest; every Grafana-stack image in the compose file except Tempo is `latest`.
14. ✅ **Node idle CPU bug:** currency and payment spun a `setTimeout(0)` loop to measure event-loop lag and sat at ~25 % CPU while idle — that would have kept an HPA scaling out. Replaced with `monitorEventLoopDelay` (≈1 %).
15. ✅ **`cart-service` build was broken by two things:** an ineffective `.dockerignore` (host `src/obj` leaked into the image build) and new advisories on OpenTelemetry .NET 1.15.0 (`TreatWarningsAsErrors` + NuGet audit). Fixed with a root `.dockerignore` and 1.15.3 packages.
16. **`cart-service` data model:** no TTL, `EmptyCart` overwrites, `AddItem` isn't atomic. Redis has no auth (the network policy isolates it).
17. **`product-catalog` cache is per pod**; refresh with `SIGUSR1` (`kubectl exec … -- kill -USR1 1`, which needs the break-glass annotation on this platform) or restart.
18. **pprof exposure:** shipping always serves `/debug/pprof` on its (unexposed) debug port; checkout/product-catalog when `ENABLE_PROFILING=1`. The charts do not expose these ports.
19. **`shipping.ShipOrder` crashes the process on a nil address** (no recovery interceptor) — latent, only `checkout` calls it with a validated address.
20. **`ad-service` returns *all* ads per matched category**; `ENV_PLATFORM` isn't read; `checkout` does not require downstreams at boot; cart logs aren't JSON by default.
21. **`shopping-assistant-service`** is GCP-only, unwired, no health endpoint/telemetry — excluded from the charts, not a Git repository.
22. **Pyroscope Java agent needs an executable `/tmp`**; a `noexec` tmpfs makes `ad-service` crash at startup (an `emptyDir` is fine).
23. **Idle telemetry noise on EKS:** with no collector in the cluster, every service logs OTLP exporter errors. Harmless, but do not mistake it for a fault.

## Admission control (Kyverno)

- **CEL: an absent map key is an error, not `false`.** `labels['x'] == 'y'` on an object without that label errors; with `failurePolicy: Fail` that turned ordinary namespace updates into denials in a first draft. Always `'x' in labels && labels['x'] == 'y'`. The unit tests cover it.
- `image()` normalises references (`mladenovskistefan/x` → registry `index.docker.io`; `redis` → `library/redis`), but `matchImageReferences` globs match the string **as written in the pod**.
- **Docker Hub rate limits and signature checks.** `pd-verify-images` reads the manifest and the `sha256-<digest>.sig` from Docker Hub at every pod admission. Anonymous pulls are limited (100 per hour per IP; on AWS all nodes share the NAT gateways' addresses), and for an enforcing policy a lookup *error* is a denial (`TOOMANYREQUESTS`), together with the nodes' own image pulls. Hence: background re-verification off, and a read-only Docker Hub token (`kyverno/dockerhub-read`, from Parameter Store through External Secrets). A registry without anonymous limits (ECR pull-through) is the cleaner long-term answer.
- **Server-side apply merges set-lists across field managers.** A leftover `kubectl apply` made a policy show `[Deny, Audit, Warn]` after Argo CD changed it. Change policies through Git only; if you must patch live, re-apply the rendered chart with `--server-side --force-conflicts --field-manager=kubectl`.
- **Exceptions** live only in the `kyverno` namespace and are matched with CEL on the pod, never with `spec.images` (before Kyverno 1.19.1 that field was ignored and the exception disabled verification for the whole pod).
- A `GeneratingPolicy` needs RBAC for what it creates (the CiliumNetworkPolicy is added in the Kyverno values); the reports controller needs RBAC for the kinds policies validate ("not ready … missing permissions"). `kubectl get vpol` shows `READY` empty for a while after creation: read `status.conditionStatus.ready`.
- The LimitRange generated by the namespace baseline fills requests/limits **before** validation, so "no resources" is allowed in `platform-demo` by design.
- Mutations touch Pods only (autogen off), so Deployments stay as rendered and Argo CD sees no drift. Webhooks skip `kube-system`, `kube-public`, `kube-node-lease`, `cilium-spire`, `cilium-secrets` and `kyverno`: a Kyverno outage must never stop the CNI or SPIRE. For the same reason Kyverno does not see a Service of type LoadBalancer created in `kube-system` (a deny rule for platform namespaces is on the backlog).
- **Webhooks and the EKS control plane.** The pod CIDR (overlay) is not routed in the VPC, so the control plane cannot reach a webhook that runs in a pod network. Kyverno's admission controller therefore uses host port 9443 and the External Secrets webhook 8443 (+ 8082/8083); 8080 belongs to the Gateway. The `external-secrets` namespace is Pod Security `privileged` for that reason.
- A new policy starts in `Audit`, is read through `kubectl get polr -A`, and flips to `Deny` when clean. Pod Security (`restricted` label) stays the first line for privileged/host settings; Kyverno deliberately does not duplicate it.

## Network identity (Cilium, SPIRE)

- Cilium identifies a pod by its labels, so **who may set `app.kubernetes.io/name` is a security control**; Kyverno enforces it.
- **SPIRE:** the server must run as root with the Cilium chart (symptoms of the wrong setup: `Authentication required` drops, `no SPIFFE ID`, `Found 0 entries`); SPIRE 1.15.3 hangs silently on a bind error, 1.12.4 prints it. A brand-new identity needs 10-15 s before it can talk, so short `Authentication required` drops after a rollout are normal. Mutual authentication authenticates but does not encrypt: WireGuard does the encryption.
- On a fresh cluster Cilium is installed before nodes and SPIRE; a one-off Job restarts Cilium once SPIRE runs. Possible deadlocks (SPIRE vs. operator) and the WireGuard kernel support on the node AMI are unconfirmed; the switches are `cilium_mutual_authentication_enabled` and `cilium_encryption_enabled` in Terraform.
- **Hubble UI has no login of its own**: the ALB's address allow-list is its only protection. The ALB must also admit the NAT gateways' public addresses, because Argo CD and Backstage fetch `https://dex…` from inside the cluster through the ALB.

## GitOps (Argo CD)

- `controller.diff.server.side` (needed so Gateway API routes and StatefulSets don't stay OutOfSync) takes effect only after a controller restart.
- HPAs without metrics-server make an app Degraded; `managedNamespaceMetadata` labels only land on a sync (trigger one with `kubectl -n argocd patch application <a> --type merge -p '{"operation":{"sync":{}}}'`).
- Sync waves gate on the health of child Applications; Applications generated by an ApplicationSet are not ordered against each other and retry until their dependencies exist.
- Terraform installs Argo CD and the root Application, then Git adopts them. Chart version, release name and namespace must match in both places, and the AppProject `platform-admin` and `platform-root` are defined in both (keep them identical). A first apply on an existing cluster replaces the nodes if the launch template changed.
- A unit test cannot find a wrong Argo CD or Dex API assumption, only a live cluster does (for example, the permission-check REST path is `applications/<action>/<project>%2F<name>`).
- With automatic sync a rollback in a UI is undone by self-heal: a rollback is a revert in Git.

## Secrets and AWS

- Nothing secret is in Git or in Terraform state. Typed secrets are SecureString parameters created in the console (a `aws_ssm_parameter` resource would put the value in the state and delete it with the platform); External Secrets reads them. Sealed Secrets was rejected: the cluster is rebuilt often and its key would be lost.
- The RDS-managed master password lives in Secrets Manager under a name like `rds!db-<random>` whose suffix changes with every database: never hard-code it, find it by prefix. It rotates every 7 days and the schedule cannot be changed; Reloader restarts the portal when the Secret changes.
- A saved Terraform plan can hold secret values and artifacts of a public repository are public, so plans are stored encrypted in the state bucket, never as artifacts. The plan role is read-only and cannot read secret values.
- IMDSv2 hop limit 1 on the nodes: pods cannot reach the instance metadata; anything that needs it (EBS CSI) uses IRSA. EBS CSI with that limit has not been run yet.

## Supply chain and releases

- **Docker Hub has no OCI referrers API**, so signatures are the classic `sha256-<digest>.sig` format (`--new-bundle-format=false`); the tag listing API may omit `.sig` tags, so use `cosign verify` to decide whether a digest is signed.
- **`v*` tags are immutable** (ruleset): a wrong or failed release is fixed by the next version. The release workflow must pass the same Trivy ignore file as CI; time-boxed ignore entries (`expired_at`) force a re-check.
- Every action and tool pin is the newest release at least 7 days old; every `uses:` is a full commit SHA; `gradle/actions` is held at v5; `actionlint` needs shellcheck installed locally or it passes things CI rejects.
- Docker Hub push secrets are per repository, not organisation-wide. Dependabot's ignore rules must be in `dependabot.yml`: Dependabot's stored `@dependabot ignore` choices are wiped by a repository transfer (that flooded the organisation with ~25 PRs once).
- A repository transfer keeps branch protection, rulesets, tags, topics, Actions secrets, open PRs and redirects, but resets secret scanning + push protection and Dependabot's stored choices.
- The free GitHub plan limits concurrent CI jobs across the organisation: roll cross-repository changes out one repository at a time.

## Developer portal (Backstage, Dex, TechDocs)

- **Dex v2.46.0 deadlocks with `storage.type: memory` (and SQLite)** on the first refresh token (dexidp/dex#4209): the whole of Dex hangs while its health checks stay green, `/auth` never answers and the portal asks for a login after every refresh. The chart uses `storage.type: kubernetes` (custom resources in `sso`, with a Role and a ClusterRole that may create the CRDs). Revisit when a fixed release exists.
- Backstage's config rejects empty strings (omit empty secret values, mark env vars optional); the OIDC provider takes `additionalScopes` (not `scope`) and `offline_access` is needed for a refresh token; arrays in config layers are replaced, not merged.
- **TechDocs strips `<link>` and `<style>`** from documentation, so `extra_css` is useless; the shared look is injected by a portal addon. TechDocs uses the external builder: nothing publishes sites on EKS yet (needs a CI step and an S3 bucket), so the Docs tab is empty there.
- The third-party Argo CD plugin's tab crashes inside entity tabs (material-table `scrollWidth of null`); its cards work. Our plugins use plain MUI tables. Its history card drops the oldest entry when `argocd.revisionsToLoad` is unset (a single deployment then shows nothing): set it.
- A frontend plugin on the new frontend system needs `signals` (frontend + backend) when the user-settings module is present; the Kubernetes plugin in a pod needs no token in config. A `yarn.lock` seed references a missing `got` patch: pin it with `resolutions`. Pack plugin tarballs **after** `yarn build`, or stale code is shipped. The portal image build needs about 2 GB of RAM.
- The portal acts as the signed-in person towards Argo CD (the Dex ID token is forwarded; Argo CD needs `allowedAudiences: [argocd, backstage]`), is read-only by decision, and reads everything else with its own identities.

---
