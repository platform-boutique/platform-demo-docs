# Handoff - portal (Backstage) with a Deployments tab (Argo CD as the signed-in person), single sign-on (Dex + GitHub), lean local profile; repositories published, PRs open and NOT merged

Written 2026-10-18 (end of the seventh session; **section 0.00 is the newest**, 0.0 is from the sixth, the rest from the fifth; where they disagree, the newest wins). Older handoffs are in `history/`. Read this first, then `README.md` (master reference; §11 repository map and workspace layout, §12 next steps, §13 CI/CD, network and admission control), then the README or the TechDocs of whatever you touch. The portal's own documentation (architecture, SSO, catalog, plugins, operations) is the `docs/` folder of `platform-demo-backstage` and is rendered in the portal itself.

**The local cluster is destroyed at the end of the session** (on request). `LOCAL_SOURCE=1 deploy/platform-demo-helm/kind/up.sh` rebuilds everything, **including the portal**, from the working tree in ~10-15 minutes (build the portal image first, see section 9). **Nothing from this session is pushed or merged**: all of it sits on local branches (section 0.1).

## 0. Do these first

### 0.00 Session 7 (2026-10-18): published, Deployments tab built and verified, lean profile without Kyverno

**Published (the maintainer lifted "local only").** 15 Dependabot PRs closed. Branch `feat/backstage-catalog` pushed and **one PR per existing repository, none merged**: workflows #20, helm #14, observability #11, `.github` #1, docs #6, app-frontend #12, ad-service #21, cart #16, checkout #13, currency #32, email #17, payment #32, product-catalog-db #9, product-catalog-service #12, recommendation #14, shipping #11. All green except: **helm #14** (Cosign + Kyverno policy tests: `platform-demo-backstage` has no digest in Git until the portal's first release - expected; Trivy passed after replacing wildcard RBAC) and **ad-service #21** (a new HIGH CVE in the Alpine base image, `libexpat` 2.8.5-r0 -> 2.9.0-r0: rebuild the image on a patched base; unrelated to the PR). Five NEW repositories created (public, category `platform`) and `main` pushed (it still carries the first, action-capable version of the plugin until PRs plugin #1 and portal #1 merge): `platform-demo-backstage` (branch renamed master -> main), `-plugin-hubble`, `-releases`, `-policies`, **`-plugin-argocd`**; guardrails applied like the others (secret scanning + push protection, squash only, branch protection with admins, tag ruleset). Their required checks are `secrets / Secret Scan` and `workflows / Workflow lint` (confirmed on the plugin-argocd PR; check the other four on their first PR). Order to merge: workflows, observability, `.github`, docs, apps; **helm last**, after the portal image exists (below).

**GitHub group fixed:** team `maintainers` created (closed) with the maintainer; the OAuth App is approved for the org; Dex now logs `groups=[platform-boutique:maintainers]`, so GitHub login makes the person an Argo CD admin. The demo user stays admin locally.

**Deployments tab (new repo `platform-demo-backstage-plugin-argocd`, packages `@platform-boutique/backstage-plugin-deployments(-backend)`, plugin id `deployments`) - READ-ONLY by the maintainer's decision.** On every entity with `argocd/app-name`: health, sync, live revision, sync policy, last operation, history with commit links, owned resources with health, images, and a link "Open in Argo CD". **It acts as the person:** the browser sends the Dex ID token in `x-dex-id-token` (the plugin asks for the portal's `internal.auth.dex` API by id), the backend forwards it as `Authorization: Bearer` to Argo CD, whose RBAC decides what they see. Needs `allowedAudiences: [argocd, backstage]` in Argo CD's `oidc.config` (done in `gitops/argocd/values.yaml`, helm #14). No generic proxy: two GET routes, names validated; a test asserts no route changes anything. **Decision:** sync, refresh and rollback are NOT in the portal. A first version had Refresh/Sync/Rollback buttons (built and tested: dry-run and real sync from the UI were recorded by Argo CD as initiated by the signed-in person) and was removed again: with automatic sync on (kept on in dev: a merge is the deploy; for EKS prod consider auto-sync without auto-prune or manual sync) Argo CD refuses rollbacks and self-heal undoes manual changes, so the real rollback is a revert in Git, and Argo CD's own UI stays for the rest. PRs: plugin repo #1, portal #1.

**History card bug (Roadie plugin) fixed:** the "ArgoCD history" card disappeared for an application with a single deployment because the plugin does `history.slice(0, argocd.revisionsToLoad || -1)`, which drops the OLDEST entry when the option is unset. `argocd.revisionsToLoad: 10` in `app-config.yaml` fixes it (verified: the card shows on `currency-service`).

**Lean profile workloads:** `PROFILE_WORKLOADS=currency-service` (comma list, remembered) deploys chosen workloads into the lean cluster; the staged values set `networkPolicy.cilium.authentication: disabled` (no SPIRE). `currency-service` is running as the demo workload (`dev-currency-service`); add more with `PROFILE_WORKLOADS=a,b kind/local-source.sh sync`.

**Lean profile** (`PROFILE=portal`) now also leaves out Kyverno and the policies: Dex + Argo CD + Backstage + gateway = 5 Applications (SPIRE/authentication already off). The Policies tab is empty there. Dex's namespaced Role lists its resources explicitly (Trivy KSV-0112): needs `useridentities` and `authsessions` as well (found at runtime; the unit of work to remember: **a unit test cannot find a wrong Argo CD / Dex API assumption, only the live cluster did**: the can-i REST path is `applications/<action>/<project>%2F<name>`).

**Catalog:** `backstage.io/source-location` on every component (the "View Source" icon), duplicate `Repository` link removed; `argocd/app-name` on dex, argocd, kyverno entities.

**Build facts:** the portal image must be built with the cluster down (cycle: pack plugins **after `yarn build`** -> copy tarballs to `vendor/` -> `yarn install --mode=update-lockfile` -> `docker build` -> `kind/up.sh`). Packing without rebuilding ships stale code (it happened once). The helm commits `allowedAudiences`, Dex RBAC and `PROFILE_WORKLOADS` are pushed to PR #14. The new repos' protected `main` needs PRs; their required checks `secrets / Secret Scan` and `workflows / Workflow lint` were CONFIRMED on plugin PR #1. The headless sign-in script must click "Log in with Email" on Dex's connector chooser.

**Cluster state at the end:** the lean cluster is RUNNING (the maintainer wants to look at it); `kind/down.sh` destroys it. Old unrelated kind cluster `platform-demo` (exited since March) exists; not touched.

**Terraform / EKS (new repo `infra-terraform`, local `infra/terraform`, public, category `infra`, guardrails applied; PR #1 open) - WRITTEN, VALIDATED, NEVER APPLIED (no AWS credentials were available).** `bootstrap/` (S3 state bucket: versioned, KMS customer key, TLS-only, native locking) and `envs/platform/` composing five modules: `network` (VPC 10.0.0.0/16, public 10.0.0.0/24+10.0.1.0/24 for the ALB only, private 10.0.10.0/24+10.0.11.0/24, one NAT gateway per zone, us-east-1a/b), `eks-cluster` (EKS 1.35, no add-ons, no nodes, public API restricted to your /32, access entries, KMS), `cilium` (Gateway API CRDs pinned + checksummed, then Cilium 1.20.2: VXLAN overlay, pod CIDR 10.244.0.0/16, kube-proxy replacement, `gatewayAPI.hostNetwork`, Hubble), `eks-nodes` (managed SPOT node group, 2 x 4 vCPU/16 GiB types m6i/m6a/m5/m5a/m7i/m7a.xlarge, AL2023, CoreDNS + EBS CSI via IRSA), `alb` (the ALB, target group on instance port 8080, security group rules, attached to the node group's Auto Scaling group). **Order matters: cluster -> Cilium -> nodes** (nodes are not Ready without a CNI), hence three modules. **No AWS resource is created by Kubernetes:** Cilium's host-network Gateway makes no LoadBalancer Service, so `terraform destroy` removes everything; leftovers can only be EBS volumes behind PVCs (find them with the tag `ebs.csi.aws.com/cluster=true`) and a Service of type LoadBalancer someone creates (add a Kyverno deny). What was checked: `terraform validate`, `fmt`, a mock-provider plan (network, cluster, ALB: 49 resources) and isolated plans of the Cilium (13) and nodes modules; Trivy locally. **Not checked (needs AWS): the real plan and apply**, that the node group becomes Ready with Cilium installed first, EKS 1.35 with Cilium 1.20.2, the ALB attachment to the managed node group's ASG. **Trivy CI job (not required) fails on 4 findings by design:** AWS-0040 public API endpoint (restricted to /32), AWS-0053 public ALB, AWS-0054 HTTP listener (no domain, no certificate), AWS-0104 node security group egress (EKS module default). Suppress inline with a comment, or decide otherwise: not done without the maintainer's say. **Needed from the maintainer to apply:** AWS account access (SSO), AWS CLI v2, the public IP as /32 in `terraform.tfvars`. **Helm repository follow-ups for an EKS env:** Gateway `listeners.http: 8080` and an optional HTTPS listener; `networkPolicy.cilium.authentication: disabled` until SPIRE is brought up on EKS; routes are host-based, so without a domain the ALB's own DNS name matches nothing (hosts-file entries or the Host header); Argo CD must be bootstrapped on the cluster (not in Terraform yet); Kyverno policy denying Service type LoadBalancer.

**Next:** (1) the maintainer looks at the tab in a browser (trust `kind/certs/ca.crt`); (2) check the read-only user (403 for an Application they may not see); (3) CI/CD tab (GitHub Actions for the service's repository + the Releases plugin), observability links and alert status (Grafana needs the full profile or EKS), status strip on the overview; (4) roll the annotations out to the 11 services; (5) first portal release v0.1.0 -> promotion PR (digest) -> merge helm #14; (6) keep Argo CD's UI (rollback, sync, break-glass) behind the gateway; (7) a shared `backstage-plugin-ci` workflow (plugins have only secret-scan + workflow-lint).

### 0.0 Session 6 (2026-10-18, local only): lean profile, Dex fix, GitHub login; where we stopped and what is next

**State at the end:** the cluster was destroyed on request (`kind/down.sh`). Everything below is in the working trees, **uncommitted** (no pushes, nothing merged). Rebuild: `PROFILE=portal LOCAL_SOURCE=1 kind/up.sh` (portal image `platform-demo-backstage:dev` is still in the local Docker image store; ~10 min; then add `backstage` / `dex` to hosts if missing).

Done and verified this session:
- **Lean profile** (section 6.0): `PROFILE=portal` = Dex, Argo CD, Backstage, Kyverno + policies, gateway; no shop, no monitoring, no SPIRE. 7 Applications, node container ~3.65 GiB.
- **Dex deadlock fixed** (section 9, first gotcha): Dex v2.46.0 hangs on the first refresh token with `memory` storage; the chart now uses `storage.type: kubernetes` (new `dex/templates/rbac.yaml`). This was the cause of "the popup is slow / never opens" and "asks to log in again".
- **GitHub login works** through Dex (OAuth App created by the maintainer, callback `https://dex.platform-demo.test/callback`, "Expire user access tokens" off). `kind/local-source.sh` turns the connector on in the staged copy when `sso/dex-secrets` holds a client ID; the demo user stays on beside it. **The client ID/secret live only in the cluster Secret: after a rebuild run `GITHUB_OAUTH_CLIENT_ID=... GITHUB_OAUTH_CLIENT_SECRET=... kind/sso-secrets.sh` then `kind/local-source.sh sync` (keep them in a password manager; never in Git or chat).** A new cluster also creates a new demo password (`kind/certs/demo-password`).
- **Windows browser:** trust `kind/certs/ca.crt` once (`certutil -user -addstore Root <path>` from PowerShell), otherwise incognito popups land on a certificate warning. Dex now shows a connector chooser, so scripted sign-ins must click "Log in with Email" first.
- Backstage chart: `config.overlay` value (extra app-config layer without an image rebuild; arrays are replaced, not merged).
- Changed files (all uncommitted): helm repo `kind/{up,local-source,sso-secrets,sso-bootstrap}.sh`, `kind/README.md`, `backstage/` and `dex/` charts, `docs/sso-and-portal.md`; `platform-demo-backstage/tools/techdocs/build-docs.sh`; this file.

Open findings to look at:
- GitHub login gave Dex `groups=[]` for the maintainer: the GitHub team `maintainers` is missing, the person is not in it, or the org has not approved the OAuth App for org data. Until fixed there is no admin in Argo CD through GitHub (demo user is admin) and ownership in Backstage comes from the catalog only.
- The portal's sign-in page has `auto` set: it tries a popup on page load without a click (browsers may block it). Removing it needs an image rebuild.
- Entities `dex`, `kyverno`, `argocd`, gateway (in `catalog/platform.yaml`, baked into the image) have no `argocd/app-name` annotation, so the Argo CD cards do not show on them. The cards work where the annotation exists (verified on `platform-demo-backstage`: overview + history).
- AWS decisions discussed (see 6.3): keep Dex for app SSO (IAM Identity Center only for AWS access), cert-manager + Let's Encrypt DNS-01 (Route53) for certificates, real domain first, ExternalDNS.

**(Done in session 7, see 0.00.) Was: Argo CD inside the portal like TechDocs - see what is deployed and sync/refresh/roll back from a service's page.** Agreed design: a new plugin in its own repo (`platform-demo-backstage-plugin-argocd`, like hubble/releases/policies) with a **Deployments tab** per entity: Application sync/health, resource tree with health, revision history, diff Git vs live; actions Refresh, Sync (prune/dry-run options), Rollback to a history revision, Restart; no create/delete (Git creates apps). **Authorization = the person's own identity, not a shared token:** the frontend gets the Dex ID token (`dexAuthApiRef` is an `OpenIdConnectApi`, `getIdToken()`), a small backend proxy forwards it to Argo CD's API, Argo CD enforces its RBAC (`platform-boutique:maintainers` = admin, everyone else read-only) and records the real person in the history. Needs `allowedAudiences: [argocd, backstage]` in `oidc.config` (`gitops/argocd/values.yaml`). Do NOT use the Roadie backend's `/sync`/create/delete endpoints as they are (one shared token, Backstage permissions are allow-all). Staged build: (1) read-only tab, (2) refresh + sync, (3) rollback. Caveat to show in the UI: auto-sync with self-heal puts a rolled-back app back to Git, a real rollback is a revert in Git. The portal image build needs ~2 GB: build with the cluster down. Test on platform apps (`dev-dex`, `dev-backstage`, `kyverno`) in the lean profile; add the `argocd/app-name` annotations first. Ask the maintainer before long autonomous steps, as always.

### 0.1 What happened and where it is

The maintainer asked for the Backstage part, done entirely locally (no pushes, no GitHub job runs), with single sign-on for every UI. Result:

| Where | Local branch | Contents |
|---|---|---|
| `platform/platform-demo-backstage` (**new repo, no remote yet**) | `master` | the portal: configuration layers, backend modules (Dex sign-in resolver), TechDocs style addon, Dockerfile, shared catalog (`catalog/`), TechDocs tooling (`tools/techdocs/`), pipeline files, its own docs |
| `platform/platform-demo-backstage-plugin-hubble` (**new repo**) | `main` | custom plugin: Hubble/Cilium flows (Network tab and page) |
| `platform/platform-demo-backstage-plugin-releases` (**new repo**) | `main` | custom plugin: what Argo CD runs vs. what the Helm repository declares (Releases tab and page) |
| `platform/platform-demo-backstage-plugin-policies` (**new repo**) | `main` | custom plugin: Kyverno findings per workload + policy catalog (Policies tab and page); replaces the TeraSky plugin |
| `deploy/platform-demo-helm` | `feat/backstage-catalog` | charts `dex/` and `backstage/`, Argo CD OIDC + portal account, Grafana OIDC, routes, Kyverno scope + allow-list, AppProject destinations, `kind/` scripts (`sso-secrets.sh`, `sso-bootstrap.sh`, `local-source.sh`, `local-source-server.py`, `load-image.sh`, `techdocs-publish.sh`, `up.sh` changes), TechDocs |
| each of the 11 `apps/*`, `platform/platform-demo-workflows`, `platform/platform-demo-observability`, `platform/org-profile` (`.github`), `docs/` | `feat/backstage-catalog` | `catalog-info.yaml`, `mkdocs.yml`, `docs/` (TechDocs); in `docs/` also this handoff and the README update |

To publish: create the four new repositories in the organisation (category `platform`), push, open one pull request per existing repository **one at a time** (free-tier CI concurrency), and follow section 0.3.

### 0.2 Prove it again

1. **Build the portal image first, with no cluster running** (`docker build -t platform-demo-backstage:dev platform/platform-demo-backstage`, ~4 min; next to a running cluster it swaps for 10+ minutes or fails, section 9).
2. `LOCAL_SOURCE=1 deploy/platform-demo-helm/kind/up.sh`. Add `backstage` and `dex` `.platform-demo.test` to the Windows hosts file and WSL `/etc/hosts` (same line as the others).
3. https://backstage.platform-demo.test, sign in as `demo@platform-demo.test` (password in `kind/certs/demo-password`). Check section 7c for what was verified.

### 0.3 Before anything can be merged

1. **GitHub OAuth App** (web UI only): callback `https://dex.platform-demo.test/callback`; then `GITHUB_OAUTH_CLIENT_ID/SECRET=... kind/sso-secrets.sh`, set `github.enabled: true` / `demoUser.enabled: false` in `envs/dev/values/dex.yaml`. Create the GitHub team `maintainers` (its slug becomes the group `platform-boutique:maintainers`: Argo CD admin, Grafana Admin, owner of every catalog entity).
2. **Catalog from GitHub:** switch `config.files` in `envs/dev/values/backstage.yaml` from `app-config.kind.yaml` to `app-config.github.yaml`, put a GitHub token or App into `GITHUB_TOKEN` of `backstage-secrets`; the Releases plugin's `valuesUrl` and `latestTag` then default to GitHub.
3. **First release of the portal image:** tag `v0.1.0` in `platform-demo-backstage` (pipeline never ran), then a promotion PR sets `images.backstage.digest` in `envs/dev/values/backstage.yaml`. Until then the digest is empty in Git and the Helm CI gate (digest + cosign for our images) will fail on it: expected.
4. **Plugin packages:** the three plugins are consumed as tarballs from `platform-demo-backstage/vendor/` until there is a package registry.
5. **TechDocs in CI** (production): `techdocs-cli generate` + `publish` to object storage (needs the AWS phase). Local publisher only for now.
6. Still open from the previous session: Docker Hub read-only token (verify-images still audits), token hygiene (the `gh` PAT), Dependabot `ignore:` rollout, first release under the org, org hygiene (see `history/HANDOFF-previous-kyverno-org-move.md` §0).

## 1. Where things stand

| Area | State |
|---|---|
| Code | 16 repos in the organisation `platform-boutique` (unchanged on GitHub) **plus 4 new local repos** (portal and its three plugins) |
| Delivery, networking, admission control, observability | unchanged (Argo CD 3.5, Cilium 1.20.2 kube-proxy-less, Kyverno 1.19.1 with 16 policies); the 29 live admission cases pass on a cluster built from the working tree |
| **Single sign-on (new)** | **Dex v2.46.0** (`dex/` chart, namespace `sso`) is the OIDC provider for Backstage, Argo CD and Grafana. GitHub connector (org `platform-boutique`, teams as groups `<org>:<team>`) configured and **off**; built-in **demo user** on in dev. Pods reach `https://dex.platform-demo.test` through a CoreDNS rewrite; they trust the local CA through ConfigMap `platform-demo-ca` |
| **Developer portal (new)** | **Backstage 1.55** (new backend + frontend system), PostgreSQL 17, read-only ServiceAccount; namespace `backstage`. Plugins: catalog, API docs, TechDocs (+ Mermaid addon, own style addon, external builder), Kubernetes, Argo CD cards (Roadie), Grafana (community), search, org, and three own plugins: **Hubble/Cilium**, **Releases**, **Policies**. Sign-in only for people who exist as `User` entities |
| Catalog | every repo has `catalog-info.yaml` (+ API entities with their `.proto`); shared Domain/Systems/Resources in `platform-demo-backstage/catalog/platform.yaml`; users/groups from GitHub in production, `catalog/org.yaml` locally |
| Documentation | every repo has TechDocs (`mkdocs.yml`, `docs/`), generated once from the READMEs by `tools/techdocs/scaffold*.py`, now hand-edited files. The shared look is injected by the portal (TechDocs strips `<link>`/`<style>`) |
| Deploy without pushing | `LOCAL_SOURCE=1 kind/up.sh`: `kind/local-source.sh` serves a staged copy of the Helm repository (Git smart HTTP via `local-source-server.py`) and the catalog descriptors on the kind network's gateway address; Argo CD reads that |
| Environment | needs docker, kind, kubectl, helm, openssl, python3 (+ `bcrypt` or `htpasswd`), node/yarn for the portal build, **~6.5 GB RAM** |

## 2. Workspace layout (local; each folder below `apps/`, `deploy/`, `platform/` is its own Git repo)

```
platform-demo/
  apps/      ad-service cart-service checkout-service currency-service email-service frontend payment-service
             product-catalog-service recommendation-service shipping-service product-catalog-db
             shopping-assistant-service (dormant, GCP-only, no repo)
  deploy/    platform-demo-helm      charts (incl. dex/, backstage/), envs/<env>/, gitops/, policies/, kind/
  platform/  platform-demo-workflows (reusable CI/CD + tools/), platform-demo-observability (compose + dashboards),
             org-profile (= .github), platform-demo-backstage, platform-demo-backstage-plugin-{hubble,releases,policies}
  infra/     (empty) future terraform/ for the platform cluster and crossplane/
  docs/      platform-demo-docs      README.md (master reference), HANDOFF.md, OTLP-MIGRATION-PLAN.md, history/
  platform-demo.code-workspace       (does not list the four new repos yet)
```

Convention for new things: services -> `apps/`; anything that provisions clouds/clusters -> `infra/`; portal, plugins, tests/monitors/alerts, AI tooling -> `platform/`; environment state only in `deploy/platform-demo-helm/envs/`; docs -> `docs/`.

## 3. Access and conventions

- **CI concurrency (free plan): never start CI in several repos at once**; roll out cross-repo changes one repository at a time. Nothing in this project may depend on paid GitHub features (Dex, Backstage and the plugins are free/open source).
- `gh` is logged in as `mladenovskistefan111` with a classic PAT (`repo`, `workflow`, `admin:org`, `project`, 7-day expiry); delete it when done. Tools in `platform-demo-workflows/tools/` default to the owner `platform-boutique`.
- Squash merges only; commit trailer `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`, PR bodies end with the Claude Code line. `gh pr edit` is broken; use `gh api -X PATCH repos/<o>/<r>/pulls/<n> -f title=... -f body=...`.
- **The permission classifier blocks** merging PRs, destructive git and anything that weakens a scan without an explicit rule. Ask first.
- User preferences: validate through the **remote pipeline** (when pushing is allowed again); **this session was explicitly local-only**; wants things tested and fixed properly; READMEs kept truthful; asks to be questioned *before* long autonomous work; prefers existing local Docker images over pulls; stop non-essential workloads under resource pressure; wants to see the running cluster before it is destroyed.
- Supply chain: every action/tool pin is the newest release at least 7 days old. Dex `v2.46.0` (2026-10-07) and `postgres:17-alpine` are pinned by digest in the charts.
- Pre-commit (gitleaks) in every clone: `pipx install pre-commit && pre-commit install`; the four new repos have `.pre-commit-config.yaml` but no hook installed.

## 4. Architecture in one screen

```
service repo --PR--> CI (workflows v2.0.2) --tag vX.Y.Z--> release.yaml --> Docker Hub (digest scanned, signed, SBOM)
                                                                              |  PR: image.digest in envs/<env>/values/<app>.yaml
platform-demo-helm (main) <-- polled every 2 min -- Argo CD ApplicationSets --> cluster
   CI: lint, render, kubeconform, Trivy, cosign gate, KYVERNO POLICY TESTS        Cilium (policies + mutual auth, Gateway)
                                                                                 Kyverno (admission)
people --GitHub--> Dex (OIDC) --> Backstage (read-only: Kubernetes, Argo CD, Grafana, Hubble Relay, GitHub) / Argo CD / Grafana
```

Image signing and the Kyverno catalog are unchanged (`history/`). **Secrets are never in Git**: client secrets, the demo password, Argo CD/Grafana tokens and the database password are generated by `kind/sso-secrets.sh` / `kind/sso-bootstrap.sh`.

## 5. Everyday procedures

- **Rebuild / tear down:** `kind/up.sh` (from GitHub `main`) or `LOCAL_SOURCE=1 kind/up.sh` (from the working tree, needs the local portal image); `kind/down.sh`.
- **After changing something under `deploy/platform-demo-helm`** in local-source mode: `kind/local-source.sh sync` (Argo CD picks it up within 2 minutes). New portal image: build it, `kind/load-image.sh platform-demo-backstage:dev mladenovskistefan/platform-demo-backstage:0.1.0`, then `sync` (it writes the digest into the staged values).
- **Documentation:** edit the repository's `docs/`; `kind/techdocs-publish.sh` renders and publishes all sites into the running portal. New repository: copy `catalog-info.yaml`/`mkdocs.yml`/`docs/` from a similar one (`tools/techdocs/scaffold.py`). The shared look is `tools/techdocs/extra.css` -> `tools/techdocs/sync-style.sh` -> rebuild the image.
- **A new plugin of ours:** own repository (`backstage-cli new`), `yarn build`, `backstage-cli repo fix --publish`, `yarn pack`, copy the tarball to the portal's `vendor/`, `yarn workspace app|backend add file:...`.
- **Release a service / add a workload / change a policy / break-glass / change a shared workflow / Dependabot:** as in `history/HANDOFF-previous-kyverno-org-move.md` §5. **Add a workload now also means** a `catalog-info.yaml` in its repo.
- **Rotate the portal's Argo CD/Grafana tokens:** `kind/sso-bootstrap.sh`.

## 6. Next work - the maintainer's decisions, in order

### 6.0 Lean profile: portal + SSO + Argo CD + Kyverno without the shop, monitoring and SPIRE (DONE and verified 2026-10-18, local only)

`PROFILE=portal LOCAL_SOURCE=1 kind/up.sh` (about 10 minutes with the portal image already built; **uncommitted** changes in `deploy/platform-demo-helm`: `kind/up.sh`, `kind/local-source.sh`, `kind/sso-secrets.sh`, `kind/sso-bootstrap.sh`, `kind/README.md`, chart `backstage/` (new `config.overlay` value); in `platform-demo-backstage`: `tools/techdocs/build-docs.sh`).

What the profile does (all in the **staged copy**, nothing in Git changes): `envs/dev/workloads.yaml` becomes `[]`, `monitoring-local` is removed from `addons.yaml`, the `shop` and `grafana` gateway routes are nulled (their namespaces do not exist), Cilium is installed with SPIRE/authentication off, `up.sh` skips the `platform-demo` namespace, the `monitoring` namespace and Grafana secrets, computes the expected Application count from the staged sources (7 here, 20 in the full profile), `sso-bootstrap.sh` skips Grafana, and the Backstage chart's new `config.overlay` (rendered to a ConfigMap, passed as a last `--config`, no image rebuild) switches the three Grafana entity cards off. **Backstage replaces arrays instead of merging them**, so the overlay repeats the base `app.extensions` entries (catalog at `/`, Argo CD tab off); keep both in step.

Verified on a fresh cluster: 7 Applications Synced/Healthy (argocd, dev-backstage, dev-dex, dev-platform-gateway, dev-policies, kyverno, platform-root); Dex discovery; Backstage sign-in through Dex (headless browser), catalog, entity pages without error banners, TechDocs (20 sites), Policies page (21 Kyverno policies); Argo CD sign-in through Dex shows the group `platform-boutique:maintainers` (admin) and all 7 Applications. Memory: the kind node container ~3.65 GiB (full stack: ~4 GiB node, ~6.5 GiB VM); no restarts.

Findings: (1) `techdocs-publish.sh` failed on the portal's own docs because `build-docs.sh` created its virtualenv **inside** the portal repository (MkDocs refuses symlinks leaving the repo): the default moved to `~/.cache/platform-demo/techdocs-venv`. (2) The Argo CD plugin logs `403` for `GET /api/argocd/find/name/dev-frontend` when the application does not exist (Argo CD answers PermissionDenied for unknown names); the card still returns an empty 200, harmless. (3) Not changed yet from the old list of Argo CD weak points: the CA through `SSL_CERT_DIR`, `policy.default: role:readonly`, `admin.enabled` (the bootstrap needs the initial admin secret), self-managed Argo CD restarting its SSO config on a values change. Not re-run in the lean profile: the 29 live admission cases (they need the shop namespace) and `kyverno test`.

### 6.1 Make the portal real
Section 0.3. Then: Hubble UI has no authentication and is the one UI outside SSO (replace it with the Network view rather than wrapping it); a `maintainers` team and CODEOWNERS on it; a GitHub App instead of a PAT; software templates (scaffolder) once repositories can be created; a Home page if wanted; the Argo CD tab (crashes, see 9) once the plugin is fixed.

### 6.2 Hubble UI -> Grafana (unchanged design, one caution)
Enable Hubble metrics, scrape into Mimir, import Cilium/Hubble dashboards, export dropped/auth flows to Loki, then disable `hubble.ui`. **The portal's Network view reads Hubble Relay**, so keep the relay (or move the plugin to metrics/Loki) when doing this.

### 6.3 Infrastructure as code (`infra/`) - unchanged
Terraform + Crossplane and the AWS list; add: S3 for TechDocs, External Secrets for the portal's Secrets, a publicly trusted certificate for Dex (then drop the CoreDNS rewrite and the CA mounts), Dex on `kubernetes`/SQL storage.

### 6.4 Observability depth, 6.5 AI, 6.6 earlier backlog - unchanged (`history/HANDOFF-previous-kyverno-org-move.md`)
New backlog from this session: no Cilium network policies for `backstage`/`sso`; Dex has no HA (its state is in Kubernetes CRDs since 2026-10-18, so it survives restarts); the portal image build needs ~2 GB RAM; no CI job type-checks/tests the plugins yet (a shared `backstage-plugin-ci` workflow is the planned fix); the Releases plugin does not show open promotion PRs or map digests to tags.

## 7. What was verified, and what was not

See **7c**. Sections 7 and 7b of the previous handoff (Kyverno, organisation move) are in `history/HANDOFF-previous-kyverno-org-move.md` and still describe the unchanged parts.

### 7c. This session

Verified on a live kind cluster built from the working tree (`LOCAL_SOURCE=1 kind/up.sh`), with a headless browser:

- sign-in at Backstage, Argo CD and Grafana through Dex (popup/redirect flows); Argo CD sees `groups: [platform-boutique:maintainers]` and Grafana grants Admin; Backstage maps the user through the catalog (the "Owned" filter works);
- catalog (19 components), entity pages with Overview (relations graph, consumed APIs, Argo CD overview/history cards, Grafana dashboards), TechDocs tab, Kubernetes tab (pod found by label selector), Network tab (live Hubble flows), Releases tab (running == declared digest);
- TechDocs of all 19 sites rendered with the shared look and Mermaid diagrams; the `extra_css` of MkDocs does not work (TechDocs strips links), the portal injects the CSS instead;
- the Policies tab (18 pass / 1 fail for the frontend: `pd-backstage-kubernetes-id` reports "mutation is not applied" on the pod, an audit finding worth a look) and the Policies page (21 policies with Deny/Audit/Mutate/Generate);
- 20 Argo CD Applications Synced/Healthy; `policies/tests/live/adversarial.sh` 29/29; the shop and all UIs answer through the Gateway.

**Final run (end of the session):** the cluster was destroyed and rebuilt from scratch with `LOCAL_SOURCE=1 kind/up.sh` using the final image; the whole bootstrap (CoreDNS rewrite, secrets, image load, local source, 20 Applications, SSO tokens, 20 TechDocs sites) ran without manual steps; sign-in at Backstage, Argo CD and Grafana, the Policies/Network/Releases tabs and the 29 admission cases were re-checked on it.

**Not verified:** the real GitHub login and GitHub catalog discovery (no OAuth App/token); the portal's release pipeline; production TechDocs publishing; the Releases plugin's GitHub tag lookup (disabled locally); `kyverno test` (49 unit cases, the CLI is not installed here) and the Helm CI after the Kyverno scope change.

## 8. Decisions taken (and why)

| Decision | Why |
|---|---|
| **Dex + GitHub** as the single sign-on provider | free, tiny, one provider for three apps; GitHub itself is OAuth only; Keycloak would cost 0.7-1 GB and a second user database |
| Standalone Dex, not Argo CD's bundled one | one issuer for everything |
| Issuer `https://dex.platform-demo.test` for browsers **and** pods (CoreDNS rewrite + Dex HTTPS listener + CA ConfigMap) | OIDC requires one issuer URL |
| Portal is **read-only** and reads with its own identities | it can show a lot and change nothing |
| Catalog from repositories, users/teams from GitHub | no hand-maintained catalog; ownership follows the organisation |
| **TechDocs: external builder, local publisher; look injected by a portal addon** | image without Python/MkDocs, no source access needed, recommended production shape; TechDocs removes `<link>`/`<style>` from documentation |
| Three **own plugins** in their own repos, tarballs from `vendor/` | Hubble, releases and policies have no working off-the-shelf plugin; a registry replaces the tarballs later |
| Own Policies plugin instead of TeraSky's | TeraSky's frontend sends entities without `kind`; the stock Kubernetes backend refuses (403) |
| Plain MUI tables in our plugins | `@backstage/core-components` Table (material-table) crashed inside entity tabs |
| Deploy locally via a staged Git remote (`local-source.sh`) | exercises the real GitOps path without pushing |
| Image loaded with `ctr images import` + explicit `<name>@<digest>` tag | `kind load docker-image` fails here; the charts pin digests |
| Backstage and Dex in **platform-scope** namespaces (audit-only Kyverno) | not workloads of the shop |
| Empty secret values omitted, env vars `optional: true` | Backstage's config rejects empty strings |
| No scaffolder, notifications, MCP actions; permissions allow-all | nothing to scaffold yet, nobody notifies, the portal is read-only |
| (earlier decisions: `history/HANDOFF-previous-kyverno-org-move.md` §8) | |

## 9. Gotchas worth remembering

- **Memory:** the kind node alone takes ~4 GiB, the API server ~1.3 GiB. A `docker build` of the portal (yarn install ~1.5 GiB) next to the full stack pushes the VM into swap (load average 300+) and can fail. **Build the image before `kind/up.sh`** (or `kind/down.sh` first). Pausing Argo CD's controllers and scaling workloads down helps but Argo heals them when the controller returns.
- **Backstage config rejects empty strings**; the OIDC provider takes `additionalScopes` (not `scope`); `offline_access` is needed for a refresh token (otherwise every reload asks to sign in again).
- **TechDocs strips `<link>` and `<style>`** from documentation, so `extra_css` is useless; CSS goes in the portal addon (`packages/app/src/modules/techdocs-style`).
- **The third-party Argo CD plugin's tab crashes** (material-table `scrollWidth of null` inside entity tabs); its cards work; the tab is disabled in `app-config.yaml`. Other tables in our plugins use plain MUI.
- **Dex v2.46.0 deadlocks with `storage.type: memory` (and SQLite) on the first refresh token used by the built-in demo user** (local-password connector re-reads Dex storage inside the refresh transaction: dexidp/dex#4209, fix PR #4312 still open, no release). The whole of Dex then hangs while its health checks stay green: `/auth` never answers (the Backstage popup stays blank or "is way too slow") and Backstage asks for a login again after every refresh. Found 2026-10-18 with a goroutine dump (`docker exec <kind node> kill -QUIT <dex pid>`, then `kubectl logs --previous`). The `dex` chart now uses `storage.type: kubernetes` (CRDs `*.dex.coreos.com`, Role in `sso` + a ClusterRole that may create CRDs). Revisit when Dex releases the fix; the GitHub connector in production does not take this path but the storage stays.
- A frontend plugin on the new system needs `signals` (frontend + backend) when `plugin-app-module-user-settings` is present (storage API depends on it).
- The Kubernetes plugin in a pod needs no token in config: the backend reads the projected ServiceAccount token itself.
- `yarn.lock` seed of `create-app` references a missing `got` patch: `resolutions: { got: npm:11.8.6 }` in the portal and plugin repos. Plugin repos also need `@types/express-serve-static-core: ^4.19.0`.
- Argo CD's Git client (go-git) cannot read "dumb" HTTP Git: `local-source-server.py` speaks smart HTTP.
- `kind load docker-image` fails; use `kind/load-image.sh` (explicit `@digest` name, otherwise the kubelet tries to pull the digest).
- Dex 2.46 distroless: the entrypoint is `docker-entrypoint`, so the container args are `dex serve /etc/dex/config.yaml`; static users accept `groups`; the static user has an empty `preferred_username`, so Backstage's resolver falls back to the email.
- A headless-browser check works on this WSL2 (Playwright Chromium in the session scratchpad, `--host-resolver-rules="MAP *.platform-demo.test 127.0.0.1"`); the Backstage sign-in opens a popup, sometimes twice.
- Everything from the previous handoff's section 9 still applies (CEL absent keys, Docker Hub limits, SSA list merging, SPIRE, Argo CD server-side diff, `gh pr edit`).

## 10. Side effects on the user's machine

- New: `~/.cache/platform-demo/` (local source staging, rendered TechDocs); `kind/certs/demo-password`; `node_modules` of the four new repos; Docker images `platform-demo-backstage:dev`, `ghcr.io/dexidp/dex:v2.46.0-distroless`, `postgres:17-alpine`, `node:24-trixie-slim` (+ build cache: `docker builder prune`); `~/.cache/ms-playwright`; a Python venv and npx cache for MkDocs/`@techdocs/cli`.
- A local HTTP server (`local-source-server.py`) runs on the kind network's gateway address while the cluster is up; `kind/down.sh` stops it.
- Hosts entries for `backstage` and `dex` were added by the maintainer.
- Left alone: the old kind cluster `platform-demo`, the stale `kind-crossplane-cluster` context.
