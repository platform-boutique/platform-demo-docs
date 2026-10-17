# Handoff - Kyverno done, ready for Backstage

Written 2026-10-17 (end of the fourth session). Older handoffs are in `history/`. Read this first, then `README.md` (master reference; §11 repository map and workspace layout, §12 next steps, §13 CI/CD, network and admission control), then the README of whatever you touch (for policies: `deploy/platform-demo-helm/policies/README.md`).
**The local cluster is destroyed** (on request). `deploy/platform-demo-helm/kind/up.sh` rebuilds everything in ~8-10 minutes.

## 0. Do these first (things only the maintainer can do)

1. **Merge the 12 open pull requests** (auto mode refused to merge on its own, so nothing below is on `main` yet):
   - `platform-demo-helm` **#11** - Kyverno (all 7 checks green, incl. the new *Kyverno policy tests*). **This one matters**; until it is merged, `main` has no Kyverno and a fresh `kind/up.sh` builds the previous state.
   - README-only PRs (absolute link to the Helm chart + a note on what Kyverno enforces): `ad-service` #13, `cart-service` #13, `checkout-service` #8, `currency-service` #19, `email-service` #11, `frontend` #7, `payment-service` #18, `product-catalog-service` #7, `recommendation-service` #9, `shipping-service` #6, `product-catalog-db` #6.
   - `gh pr merge <n> -R mladenovskistefan111/<repo> --squash --subject "<title> (#<n>)"` (merged branches are deleted automatically by the repo setting).
2. **Add the new required check** to `main` protection of `platform-demo-helm` once #11 is merged: `Kyverno policy tests` (use `platform/platform-demo-workflows/tools/protect-repo.sh platform-demo-helm '<json list of all required checks>'`, take the list from the last green PR with `tools/required-checks.sh platform-demo-helm 11` - it now includes the new job).
3. **Rebuild from `main` and prove it** (this was only proven on the branch, see 7): `deploy/platform-demo-helm/kind/up.sh`, then `deploy/platform-demo-helm/policies/tests/live/adversarial.sh` (29 cases, must end `all cases passed`). Expect 18 Applications Synced/Healthy.
4. **Docker Hub read-only token** (Account settings -> Personal access tokens -> *Read-only*), then `DOCKERHUB_USERNAME=... DOCKERHUB_TOKEN=... kind/up.sh` (or create Secret `kyverno/dockerhub-read`), set `verifyImages.pullSecret: dockerhub-read` in `envs/dev/values/policies.yaml` and delete the `verify-images: Audit` line there. Without it, signature verification only reports (anonymous pulls are limited to 100/hour/IP and a throttled lookup is a denial). See `policies/README.md`, "Docker Hub and signature verification".
5. **Revoke the `gh` token** when done: https://github.com/settings/tokens (scopes `repo`, `workflow`, `read:org`).
6. **"Project" for the repos.** Done locally: the checkout is grouped by function (section 2) and `platform-demo.code-workspace` opens it all. Not done, and why: (a) a GitHub *organization* cannot be created through the API, and moving the repos into one changes every repo path - the keyless signatures' identity (`github.com/mladenovskistefan111/<repo>/.github/workflows/...`), the Argo CD `repoURL`s, the Helm CI cosign regexp, every pinned workflow reference and the v1.0.0 tags would all need a migration; do it deliberately, not casually; (b) a GitHub *Project* (board) needs the `project` token scope (`gh auth refresh -s project`, interactive). Repos are grouped on GitHub with topics (`platform-demo`, `app`/`deploy`/`platform`/`docs`).

## 1. Where things stand

| Area | State |
|---|---|
| Code | 15 repos under github.com/mladenovskistefan111 (10 services, `product-catalog-db`, `platform-demo-helm`, `platform-demo-observability`, `platform-demo-workflows`, **`platform-demo-docs`** - new, holds the docs that used to float outside Git). `main` everywhere except the 12 PRs in section 0 |
| Releases | `v1.0.0` on the 10 services, helm, observability. Docker Hub has `:1.0.0`, `:latest`, `:sha-*` (signed, SBOM attested) |
| Shared pipelines | `platform-demo-workflows` **v2.0.1**; every caller pins it by SHA. CI on pull requests only, release on `v*` tags / manual dispatch |
| Protection | `main` of all repos: PR + required checks, linear history, no force-push, enforced for admins; `v*` tags immutable; secret scanning + push protection; CODEOWNERS; Dependabot; gitleaks pre-commit hooks in every clone |
| Delivery | Argo CD 3.5 with ApplicationSets (`gitops/apps`), env config in `envs/dev`; 18 Applications (12 workloads, gateway, monitoring-local, argocd, kyverno, dev-policies, root) |
| Networking | Cilium 1.20.2, **kube-proxy-less** (`kubeProxyReplacement: true`, `kubeProxyMode: none`, no kube-proxy DaemonSet - verified), Gateway API, Hubble, SPIFFE/SPIRE mutual auth enforced |
| **Admission control (new)** | **Kyverno 1.19.1** + 16 policies/mutations/generators as a chart (`deploy/platform-demo-helm/policies/`), **enforced for `platform-demo`**, report-only for `monitoring`/`argocd`. Catalog, rollout, exceptions, break-glass: the policies README |
| Tests of the policies | `kyverno test` 49 cases; CI job *Kyverno policy tests* evaluates every rendered workload/add-on of every environment against the policies; live suite `policies/tests/live/adversarial.sh` 29 cases (each asserts the policy that denies) |
| Environment | kind cluster destroyed. Needs docker, kind, kubectl, helm, openssl, ~6 GB RAM |
| Not done | everything in section 6 |

What Kyverno enforces in `platform-demo`, in one paragraph: a pod may only carry one of the 12 workload names in `app.kubernetes.io/name` (Cilium's identity!), must run under the ServiceAccount of the same name and cannot relabel itself; images only from `mladenovskistefan/*`, `redis`, `busybox`, pinned by digest, never `:latest`, ours signature-verified (dev: audit, see 0.4); read-only rootfs, requests + memory limit, probes, no API token (mutations default the token off, seccomp, add `backstage.io/kubernetes-id`); Services ClusterIP only, HTTPRoutes only for `*.platform-demo.test`, only Argo CD/admins touch CiliumNetworkPolicy/Gateway/HTTPRoute; `kubectl exec/attach/debug` only with `platform.demo/break-glass=true` on the pod; namespaces keep Pod Security; a generated baseline (quota, LimitRange, default-deny CNP) per workload namespace.

## 2. Workspace layout (local; each folder below `apps/`, `deploy/`, `platform/` is its own Git repo)

```
platform-demo/
  apps/      ad-service cart-service checkout-service currency-service email-service frontend payment-service
             product-catalog-service recommendation-service shipping-service product-catalog-db
             shopping-assistant-service (dormant, GCP-only, no repo)
  deploy/    platform-demo-helm      charts, envs/<env>/, gitops/, policies/, kind/
  platform/  platform-demo-workflows (reusable CI/CD + tools/), platform-demo-observability (compose + dashboards)
  infra/     (empty) future terraform/ for the platform cluster and crossplane/
  docs/      platform-demo-docs      README.md (master reference), HANDOFF.md, OTLP-MIGRATION-PLAN.md, history/
  platform-demo.code-workspace
```

Convention for new things: services -> `apps/`; anything that provisions clouds/clusters -> `infra/`; portal, tests/monitors/alerts, AI tooling -> `platform/`; environment state only in `deploy/platform-demo-helm/envs/`; docs -> `docs/`. Non-service repos keep the `platform-demo-` prefix. No script depends on the old flat layout (checked); the relative README links that did were replaced by GitHub URLs.

## 3. Access and conventions

- `gh` is logged in as `mladenovskistefan111` (scopes `repo`, `workflow`, `read:org`; no `project` scope).
- Squash merges only; commit trailer `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`, PR bodies end with the Claude Code line. `gh pr edit` is broken (GraphQL deprecation); use `gh api -X PATCH repos/<o>/<r>/pulls/<n> -f title=... -f body=...`.
- **The permission classifier blocks** merging PRs, destructive git (remote branch deletion, history rewrites, force-push) and anything that weakens a scan, without the user's explicit permission rule. Ask first, or let the user merge.
- User preferences: validate through the **remote pipeline**; wants things tested and fixed properly; READMEs kept truthful; asks to be questioned *before* long autonomous work (this session: questions first, plan, then green light).
- Supply chain: every action/tool pin is the newest release at least 7 days old (`platform-demo-workflows/tools/bump-actions.py`); `gradle/actions` held at v5. Third-party downloads are verified against the upstream checksum file (the Kyverno CLI in CI is pinned by SHA-256).
- Pre-commit (gitleaks) in every clone: `pipx install pre-commit && pre-commit install` in a fresh clone.

## 4. Architecture in one screen

```
service repo --PR--> CI (workflows v2.0.1) --tag vX.Y.Z--> release.yaml --> Docker Hub (digest scanned, signed, SBOM)
                                                                              |  PR: image.digest in envs/<env>/values/<app>.yaml
platform-demo-helm (main) <-- polled every 2 min -- Argo CD ApplicationSets --> cluster
   CI: lint, render, kubeconform, Trivy, cosign gate, KYVERNO POLICY TESTS        Cilium (policies + mutual auth, Gateway)
                                                                                 Kyverno (admission: identity, images, hardening, exposure)
```

Image signing is classic cosign format (`sha256-<digest>.sig`); identity `https://github.com/mladenovskistefan111/platform-demo-workflows/.github/workflows/docker-promote.yaml@<ref>` (older images: per-repo `ci-main.yaml`). Kyverno and the Helm CI use the same regexp `^https://github.com/mladenovskistefan111/[a-z-]+/\.github/workflows/[a-z.-]+@.+$`. Secrets are never in Git (DB credentials from `up.sh`, TLS from `make-tls.sh`, optional Docker Hub read credential).

## 5. Everyday procedures

- **Rebuild / tear down:** `deploy/platform-demo-helm/kind/up.sh` / `down.sh`; add `127.0.0.1 shop|grafana|argocd|hubble.platform-demo.test` to the Windows hosts file and WSL `/etc/hosts`. Argo password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.
- **Release a service:** merge to `main`, `git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z`. **Promote:** PR changing `image.digest` in `envs/<env>/values/<app>.yaml`.
- **Add a workload:** one line in `envs/<env>/workloads.yaml` **and** its name in `policies/values.yaml` (`identity.workloads`; a new image repository goes in `images.workload.repositories`) - the Helm CI fails the PR otherwise.
- **Change a policy:** edit `policies/`, add/adjust fixtures in `policies/tests/`, `kyverno test`, then test live on a branch (point `targetRevision`/`revision` under `gitops/` at the branch in a throw-away commit and `kubectl apply -f gitops/bootstrap/root.yaml`; **revert that commit before merging**). New policies start in `Audit`; read `kubectl get polr -A`; flip to `Deny` when clean. **Never `kubectl apply` policies by hand** (server-side-apply list ownership leaves stale `validationActions`; fix in the policies README).
- **Break-glass exec:** `kubectl -n platform-demo annotate pod <p> platform.demo/break-glass=true`, exec, then remove the annotation.
- **Change a shared workflow:** `platform-demo-workflows/README.md` ("Changing a workflow").
- **Dependabot:** minor/patch grouped, majors one PR each; group PRs ignore `@dependabot ignore`: close via the API.

## 6. Next work - the maintainer's decisions, in order

### 6.1 Backstage (next session; "I will start the Backstage plan")
- Goal stated by the maintainer: **one pane of glass** for the platform. Start it as its own repo `platform/platform-demo-backstage` (own pipeline, signed image, Kyverno allow-list entry in `policies/values.yaml`, own namespace, Postgres, read-only ServiceAccount for the Kubernetes plugin; keep an eye on RAM: ~0.5-1 GB).
- Plan agreed in discussion: catalog from a `catalog-info.yaml` in every repo (annotations: `argocd/app-name: dev-<svc>`, `backstage.io/kubernetes-id: <svc>` - already set on pods by Kyverno, `grafana/...`); plugins that exist: Kubernetes (core), Argo CD (Roadie, community/Red Hat), Kyverno policy reports ([Kyverno Policy Reports](https://roadie.io/backstage/plugins/kyverno-policy-reports/) by Terasky; [Policy Reporter](https://roadie.io/backstage/plugins/policy-reporter/), beta), Grafana (`@k-phoen/backstage-plugin-grafana`: alerts list + dashboard embedding; needs `allow_embedding`, `cookie_samesite=none`; barely maintained). **Custom plugins** to write where none exists: Hubble/Cilium (flows, drops, authentication), release/promotion view (tags -> digest PRs -> Argo state), maybe policy exceptions and break-glass audit.
- Decisions: Grafana **stays** (backend for traces/logs/metrics; Backstage embeds finished dashboards). Argo CD UI and Hubble UI can be removed from daily use only after Backstage covers them; Argo CD's UI is hard to fully replace (diff, hard refresh, rollback) - keep it reachable by port-forward. SSO/OIDC for Backstage + Argo + Grafana is the real work (iframes need compatible cookie settings).
- Remove the Policy Reporter UI idea: not needed, Backstage shows policy reports.

### 6.2 Hubble UI -> Grafana (maintainer: "would be awesome")
Assessment: feasible for most of it, not for all of it. Today Hubble metrics are **disabled** (`cilium-dbg status` -> `Metrics: Disabled`). Design: (1) enable `hubble.metrics` (dns, drop, tcp, flow, port-distribution, httpV2) and scrape the agent from Alloy into Mimir (agent metrics `:9962`, Hubble `:9965`); import Cilium/Hubble dashboards into `monitoring-local` (Cilium publishes them on grafana.com; verify ids); (2) export dropped/denied/authentication flows with the static Hubble exporter (`hubble.export.static`, file -> Alloy `loki.source.file` -> Loki) so "who was dropped and why" is searchable in Grafana; (3) then set `hubble.relay.enabled=false` and `hubble.ui.enabled=false` in `kind/cilium-values.yaml` and drop the `hubble` HTTPRoute - saves the relay and UI pods (not measured; roughly 100-150 MiB). **What is lost:** the interactive service map and live flow tail in a browser; `kubectl -n kube-system exec ds/cilium -c cilium-agent -- hubble observe ...` keeps working (needs no relay). A Grafana Node Graph panel over flow metrics can approximate the map. Do this together with the Backstage Grafana work.

### 6.3 Infrastructure as code (`infra/`)
Terraform for the platform cluster (cluster, networking, state backend) and Crossplane (self-service infra; compositions), then the AWS list: RDS instead of `product-catalog-db` (delete chart + the Helm CI allow-list entry + the Kyverno exception), cert-manager + ingress/TLS, External Secrets, SPIRE storage on EBS, Cilium installed by the cluster pipeline, EKS registered in Argo CD (cluster generator), **a registry without anonymous pull limits (ECR pull-through)** so image-signature verification can be enforced, Kyverno HA (3 admission replicas + PDBs).

### 6.4 Observability depth (maintainer: synthetic tests, monitors, alerts)
Synthetic checks (shop journey through the Gateway: browse -> cart -> checkout; Grafana Synthetic Monitoring/k6 or blackbox probes), SLOs per service, alert rules in Mimir/Grafana, Kyverno and Cilium metrics (policy-report counts, admission latency, drops, auth failures). Lives in `platform/` (new repo) + `platform-demo-observability`.

### 6.5 AI
No design yet; the maintainer will add AI tooling on top of the platform.

### 6.6 Earlier backlog (unchanged)
Semantic-versioning automation (conventional-commit PR title check, release-please via a GitHub App/fine-grained token because `GITHUB_TOKEN` tags do not trigger workflows, strict `vX.Y.Z` filter, `-rc.N`), automated promotion PRs + `envs/prod`, Cilium and Gateway API CRDs under GitOps, network policies for `argocd`/`monitoring`, WireGuard if the data path must be encrypted, Argo CD SSO + rotating the initial admin secret, hardening backlog (payment logs full card data, SIGTERM handling in Node/Python, JSON logs for cart, gRPC load balancing, base images by digest, Node `.trivyignore` entries), open Dependabot PRs (cart-service #4-#9 NuGet; observability #3 redis 8, #4 tempo 3), decide a long-term home for `shopping-assistant-service`, optional unused `tests/e2e` code.

## 7. What was verified, and what was not

Verified on a live kind cluster (running the branch `feat/kyverno`, 2026-10-17): 18 Applications Synced/Healthy; the shop works end to end through the Gateway; Kyverno uses ~250 MiB (admission ~107, background ~85, reports ~52); all 29 live admission cases pass (identity theft with default SA / other SA / unknown name / relabel, foreign registry, other Docker Hub repo, tag instead of digest, probes, writable rootfs, token, privileged, NodePort, LoadBalancer, foreign HTTPRoute, a non-admin editing CiliumNetworkPolicy, namespace Pod Security removal, exec/debug with and without break-glass); mutations (token off, seccomp, Backstage label) and LimitRange defaults applied; the baseline default-deny CNP regenerates in ~3 s when deleted; before the Docker Hub quota ran out: all 10 deployed images verify, an image without a valid signature is denied, the `product-catalog-db` exception admits only that image under its own label; `kyverno test` 49/49; the CI evaluation fails when a rendered manifest violates a policy (checked with a deliberately broken manifest).
**Not verified:** a from-scratch `kind/up.sh` on `main` with Kyverno (the PRs were not merged; the cluster was destroyed afterwards on request). Section 0.3.

## 8. Decisions taken (and why)

| Decision | Why |
|---|---|
| Trunk-based, tags for releases, envs as directories | user's choice; Argo CD + digest promotion by PR |
| Squash-merge only, PR + checks, 0 required approvals | one maintainer |
| Kyverno **CEL policy types** (`policies.kyverno.io`), no `ClusterPolicy` | `ClusterPolicy` is deprecated in 1.19 |
| Policies scoped by **namespace name**, not label | a namespace cannot opt out by being relabelled |
| Workload profile enforced, platform profile (monitoring, argocd) audit-only | third-party charts; Argo CD ships no controller liveness probe on purpose (a PolicyException documents it) |
| Mutations on Pods only (autogen off) | Deployments stay as rendered, no Argo CD drift |
| Exceptions only from namespace `kyverno`, matched with CEL, never `spec.images` | Kyverno < 1.19.1 ignored `spec.images` (whole pod skipped verification) |
| `verify-images` audits in dev | Docker Hub anonymous rate limit; an error is a denial for an enforcing policy; fix = read-only token (0.4) |
| `product-catalog-db` = temporary local-test image, unsigned, allow-listed | user's decision (RDS on AWS); Helm CI list + Kyverno exception, remove both together |
| Backstage not part of the Kyverno job; Policy Reporter UI skipped | Backstage will show reports |
| Workspace grouped by function; no GitHub organization yet | org move would break signing identity, Argo repoURLs, pins (0.6) |
| Cilium CNI + Gateway API, kube-proxy-free; enforced mutual auth (Beta, authenticates but does not encrypt); SPIRE server as root; cosign 3 with classic signatures | earlier sessions, see `history/` |

## 9. Gotchas worth remembering (policies: see `policies/README.md`)

- **CEL: an absent map key is an error, not false** (`labels['x']` on an object without `x`): use `'x' in labels && ...`. A first draft would have denied Argo's namespace updates. Unit tests cover it.
- `image()` normalises refs (`index.docker.io`, `library/redis`) but `matchImageReferences` globs match the string **as written in the pod**.
- Docker Hub: anonymous pulls are limited per IP (the rate-limit preview showed `100;w=3600`, i.e. a 1-hour window); Kyverno's manifest/signature lookups hit the limit during testing (`TOOMANYREQUESTS`) together with the node's own image pulls. The tag listing API may omit `.sig` tags - use `cosign verify` to decide whether a digest is signed (`sha-8d3a1fe` **is** validly signed; the older note saying otherwise was wrong).
- Server-side apply merges set-lists across field managers: change policies through Git only.
- A `GeneratingPolicy` needs RBAC for what it creates (CiliumNetworkPolicy added in `gitops/kyverno/values.yaml`); the reports controller needs RBAC for the kinds policies validate ("not ready ... missing permissions").
- The generated LimitRange fills requests/limits before validation, so "no resources" is allowed in `platform-demo` by design.
- SPIRE 1.15 + Cilium chart: server as root; symptoms `Authentication required` / `no SPIFFE ID` / `Found 0 entries`. A new identity needs 10-15 s. After a rollout you will see short `Authentication required` drops - normal.
- Argo CD: `controller.diff.server.side` needs a controller restart; HPAs without metrics-server make apps Degraded; `managedNamespaceMetadata` labels only land on a sync (trigger one: `kubectl -n argocd patch application <a> --type merge -p '{"operation":{"sync":{}}}'`).
- `gh pr edit` fails; `kind load docker-image` fails here; musl + VPN DNS search domain needs `ndots: 2`; tags are immutable (a wrong tag means the next patch version).
- Memory on this WSL2 VM (7.5 GiB): the Kubernetes API server alone used ~1.3 GiB; whole node ~4.6 GiB with Kyverno. No resource cuts were needed for Kyverno.

## 10. Side effects on the user's machine

- `pipx install pre-commit`; `~/.cache/pre-commit`; hooks in `.git/hooks` of all clones; helm repos `argo`, `cilium`, `kyverno` added; Kyverno CLI v1.19.1 and cosign v3.1.3 were downloaded (checksum-verified) into the session scratchpad only.
- Docker images pulled during testing (Cilium, SPIRE, Argo CD, Hubble, Grafana stack, Kyverno, busybox ...): `docker image prune -a` reclaims space.
- Left alone: the old kind cluster `platform-demo` (cannot start; `kind delete cluster --name platform-demo`) and the stale `kind-crossplane-cluster` context. `kind/certs/` (local CA, git-ignored) stays.
- The local checkout was **reorganised** (section 2): old paths such as `~/Desktop/platform-demo/frontend` are now `~/Desktop/platform-demo/apps/frontend`; reopen the VS Code workspace from `platform-demo.code-workspace`.
