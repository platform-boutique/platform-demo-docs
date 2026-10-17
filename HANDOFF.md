# Handoff - the developer portal (Backstage) and single sign-on are built and tested locally, nothing is pushed

Written 2026-10-17 (end of the fifth session). Older handoffs are in `history/`. Read this first, then `README.md` (master reference; §11 repository map and workspace layout, §12 next steps, §13 CI/CD, network and admission control), then the README or the TechDocs of whatever you touch. The portal's own documentation (architecture, SSO, catalog, plugins, operations) is the `docs/` folder of `platform-demo-backstage` and is rendered in the portal itself.

**The local cluster is destroyed at the end of the session** (on request). `LOCAL_SOURCE=1 deploy/platform-demo-helm/kind/up.sh` rebuilds everything, **including the portal**, from the working tree in ~10-15 minutes (build the portal image first, see section 9). **Nothing from this session is pushed or merged**: all of it sits on local branches (section 0.1).

## 0. Do these first

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

### 6.0 Lean profile: portal + SSO + Argo CD without the shop and monitoring (requested by the maintainer, not started)

Goal: running everything (~6.5 GB) is inefficient; the portal, Dex and Argo CD should run alone (estimate ~3.5 GB: Cilium + SPIRE + Hubble ~1 GB, API server ~1.3 GB, Argo CD ~0.8 GB, Dex, Backstage and PostgreSQL ~0.7 GB; **not measured**).

**What the portal needs, and what it does without the rest**

| Needed | Without it |
|---|---|
| Cilium, CoreDNS rewrite, Gateway, the CA/TLS secrets, Dex, PostgreSQL, Argo CD (it deploys Dex and Backstage and is itself one of the three SSO apps) | not optional |
| Kyverno + `dev-policies` | optional; then the Policies tab and page are empty/error, the admission rules are not in force |
| Shop workloads (`platform-demo` namespace) | the portal, catalog, TechDocs and SSO work; the Kubernetes, Network, Policies and Releases tabs are empty ("not deployed"); the Argo CD cards have no Applications for the services |
| Monitoring (`monitoring-local`, Grafana) | Grafana SSO is moot; the Grafana cards and the `/grafana/api` proxy error; `kind/sso-bootstrap.sh` **fails** waiting for `deploy/grafana` |

**Plan** (all in `deploy/platform-demo-helm`, to be tested on a cluster):
1. `PROFILE=portal` for `kind/up.sh` / `kind/local-source.sh`: in the **staged copy only**, empty `envs/dev/workloads.yaml` (`[]`), keep `platform-gateway`, `dex`, `backstage` in `envs/dev/addons.yaml`, and optionally drop `application-kyverno.yaml` and `application-policies.yaml` from `gitops/apps/` (the files in Git stay as they are; no second environment directory, because the ApplicationSets glob `envs/*/`).
2. `up.sh`: the wait loop expects 20 Applications (`-ge 20`): derive the number from the staged `gitops/apps` and `envs` instead; do not create the `monitoring` namespace/secrets when Grafana is absent; skip the `platform-demo` namespace and DB secret for the lean profile.
3. `kind/sso-bootstrap.sh`: skip the Grafana half when `deploy/grafana` does not exist (the Argo CD token is still created).
4. Backstage: make the Grafana plugin and the `/grafana/api` proxy tolerant when Grafana is absent (disable `entity-card:grafana/*` in the overlay for the lean profile, or leave `GRAFANA_TOKEN` unset and accept the card errors); same for the cluster-dependent tabs if they should be hidden rather than empty.
5. Check the **Argo CD and SSO side in the lean profile** (the maintainer called it "not good"): login through Dex, `maintainers` admin rights (`g, platform-boutique:maintainers, role:admin`), the `backstage` account token, applications visible in the UI, the Argo CD cards in the portal; plus the known weak points: Argo CD trusts the local CA through `SSL_CERT_DIR` (replaces the system roots for that process), `policy.default: role:readonly` lets every signed-in person read everything, `admin.enabled` is still true (the bootstrap needs the initial admin secret), the `argocd` app manages itself so a values change restarts the SSO configuration.
6. Document the result in `kind/README.md`, the portal's *Run it locally* page and this handoff; measure real memory with `docker stats` and update the estimate.

Until this is done, use `kind/down.sh` + a fresh `up.sh`, or scale the shop and the monitoring backends to 0 **and pause Argo CD's controller** first (Argo CD heals them otherwise).

### 6.1 Make the portal real
Section 0.3. Then: Hubble UI has no authentication and is the one UI outside SSO (replace it with the Network view rather than wrapping it); a `maintainers` team and CODEOWNERS on it; a GitHub App instead of a PAT; software templates (scaffolder) once repositories can be created; a Home page if wanted; the Argo CD tab (crashes, see 9) once the plugin is fixed.

### 6.2 Hubble UI -> Grafana (unchanged design, one caution)
Enable Hubble metrics, scrape into Mimir, import Cilium/Hubble dashboards, export dropped/auth flows to Loki, then disable `hubble.ui`. **The portal's Network view reads Hubble Relay**, so keep the relay (or move the plugin to metrics/Loki) when doing this.

### 6.3 Infrastructure as code (`infra/`) - unchanged
Terraform + Crossplane and the AWS list; add: S3 for TechDocs, External Secrets for the portal's Secrets, a publicly trusted certificate for Dex (then drop the CoreDNS rewrite and the CA mounts), Dex on `kubernetes`/SQL storage.

### 6.4 Observability depth, 6.5 AI, 6.6 earlier backlog - unchanged (`history/HANDOFF-previous-kyverno-org-move.md`)
New backlog from this session: no Cilium network policies for `backstage`/`sso`; Dex has no HA/persistence; the portal image build needs ~2 GB RAM; no CI job type-checks/tests the plugins yet (a shared `backstage-plugin-ci` workflow is the planned fix); the Releases plugin does not show open promotion PRs or map digests to tags.

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
