# Handoff (archived) - Argo CD + Cilium + CI v2 done, ready for Kyverno

Written 2026-10-17 (end of the third session). Older handoffs are kept as `HANDOFF-previous-ci-and-helm.md` and `HANDOFF-previous-otlp-ci.md`.
Read this first, then `README.md` (master reference; §12 next steps, §13 CI/CD/CD/network), then the README of whatever you touch.
**The local cluster is destroyed** (on request). `platform-demo-helm/kind/up.sh` rebuilds everything in ~8 minutes.

## 1. Where things stand

| Area | State |
|---|---|
| Code | 14 repos under github.com/mladenovskistefan111, everything on `main` via squash PRs, **no open PRs of ours** (only Dependabot, see 6) |
| Releases | `v1.0.0` tagged on the 10 services, `platform-demo-helm`, `platform-demo-observability`. Docker Hub has `:1.0.0`, `:latest`, `:sha-*` for all ten services (signed, SBOM attested for the newer builds) |
| Shared pipelines | `platform-demo-workflows` **v2.0.1** (Go/Node/Python/Java/.NET CI, docker-build, docker-promote, secret-scan, workflow-lint). Every caller pins v2.0.1 by SHA. CI on pull requests only, release on `v*` tags / manual dispatch |
| Protection | `main` of all 14 repos: PR + required checks, linear history, no force-push, enforced for admins; `v*` tags immutable; secret scanning + push protection; CODEOWNERS; Dependabot; local gitleaks hooks (pre-commit) in all clones |
| Delivery | **Argo CD** (3.5, chart 10.10.0) with **ApplicationSets** (`gitops/apps`), env config in `envs/dev`, Argo CD self-managed. Verified from scratch: 16 Applications Synced/Healthy |
| Networking | **Cilium 1.20.2** CNI (kube-proxy replacement), **Cilium Gateway API** (Envoy Gateway removed), Hubble, **SPIFFE/SPIRE mutual authentication enforced** via CiliumNetworkPolicies from `platform-common` (`networkPolicy.type: cilium`) |
| Environment | kind cluster **destroyed**. Rebuild: `cd platform-demo-helm && kind/up.sh` (needs docker, kind, kubectl, helm, openssl; ~6 GB RAM). Tear down: `kind/down.sh` |
| Not done | Kyverno (next), semver automation, prod environment / AWS, Cilium under GitOps, the Dependabot PRs listed in 6 |

Verified on the final fresh build (2026-10-17): browse -> add to cart -> checkout through the Gateway ("Your order is complete!"); telemetry from all ten services in Mimir/Tempo/Loki; intruder pod from another namespace dropped (`Policy denied`); a pod with the allowed `frontend` label but a *new* identity dropped with `Authentication required` while SPIRE was down, and connected after ~10-15 s once SPIRE was up.

## 2. Access and conventions

- `gh` is logged in as `mladenovskistefan111` (token in `~/.config/gh/hosts.yml`, scopes `repo`, `workflow`, `read:org`). **The user must still revoke it** when done: https://github.com/settings/tokens
- Squash merges only; commit trailer `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`, PR bodies end with the Claude Code line. `gh pr edit` is broken by a GitHub GraphQL deprecation; use `gh api -X PATCH repos/<o>/<r>/pulls/<n> -f title=... -f body=...`.
- Merge a green PR: `gh pr merge N -R mladenovskistefan111/<repo> --squash --delete-branch --subject "<title> (#N)" --body "Co-Authored-By: ..."`.
- **The permission classifier blocks** destructive git (remote branch deletion without explicit approval, history rewrites, force-push) and anything that weakens a scan (allowlists, `.gitleaksignore`). Ask the user first, get an explicit yes.
- User preferences: validate through the **remote pipeline** (fix, push, dispatch/PR, read logs, repeat) rather than long local runs; wants things tested and fixed properly; wants READMEs kept truthful; asks to be questioned *before* long autonomous work.
- Supply chain: every action/tool pin is the newest release **at least 7 days old** (`platform-demo-workflows/tools/bump-actions.py`); `gradle/actions` is held at v5 (v6 licence change, not accepted for the user).
- Pre-commit hooks (gitleaks) are installed in every clone; a fresh clone needs `pipx install pre-commit && pre-commit install`. `--no-verify` skips them; GitHub push protection and CI don't.

## 3. Architecture in one screen

```
service repo --PR--> CI (workflows v2.0.1) --tag vX.Y.Z--> release.yaml --> Docker Hub (digest scanned, signed, SBOM attested)
                                                                              |  PR: image.digest in envs/<env>/values/<app>.yaml
platform-demo-helm (main) <-- polled every 2 min -- Argo CD ApplicationSets --> cluster (Cilium: policies + mutual auth, Gateway)
```

- `platform-demo-helm/` : charts (`platform-common` library, 12 workload charts, `platform-gateway`, `monitoring-local`, umbrella `platform-demo` for plain `helm install`), **`envs/dev/`** (what runs: `workloads.yaml`, `addons.yaml`, `values/*.yaml` with the v1.0.0 digests), **`gitops/`** (Argo CD values, bootstrap root, AppProjects, ApplicationSets - read `gitops/README.md`), **`kind/`** (up/down scripts, cluster + Cilium values, runbook with all gotchas - read `kind/README.md`), `.github/` (PR CI, `scripts/render-envs.py`).
- `platform-demo-workflows/` : the reusable workflows + `tools/` (bump-actions, generators for callers/language workflows/guardrails, branch-protection scripts). Read its README for the change/release procedure.
- Image signing: **classic cosign format** (`sha256-<digest>.sig` tags) because Docker Hub has no OCI referrers API; keyless identity is `https://github.com/mladenovskistefan111/platform-demo-workflows/.github/workflows/docker-promote.yaml@<ref>` (older images: `.../<service>/.github/workflows/ci-main.yaml@refs/heads/...`). The Helm CI verifies every deployed `mladenovskistefan/*` image against `^https://github.com/mladenovskistefan111/[a-z-]+/\.github/workflows/[a-z.-]+@.+$`.
- Secrets never in Git: DB credentials (random, created by `up.sh`), TLS (`make-tls.sh`, local CA in `kind/certs/`, git-ignored), Docker Hub creds (repo secrets).

## 4. Everyday procedures

- **Rebuild the environment:** `platform-demo-helm/kind/up.sh`; add `127.0.0.1 shop.platform-demo.test grafana.platform-demo.test argocd.platform-demo.test hubble.platform-demo.test` to the Windows hosts file and WSL `/etc/hosts` (the script prints it). Argo password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.
- **Release a service:** merge to `main`, then `git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z` in that repo; `release.yaml` publishes `:X.Y.Z`, `:sha-*`, `:latest`. (Manual `Release` run with `publish=true` publishes only `:sha-*`.)
- **Promote to an environment:** PR changing `image.digest` in `envs/<env>/values/<app>.yaml` (digest from `docker buildx imagetools inspect mladenovskistefan/<svc>:X.Y.Z`). CI renders, validates, verifies the signature; merge -> Argo syncs.
- **Add a workload to an env:** one line in `envs/<env>/workloads.yaml` (+ optional `values/<app>.yaml`). **Add an environment:** a new `envs/<name>/` directory (the generators match `envs/*/`); a second cluster additionally needs a destination in the list entries and the cluster registered in Argo CD.
- **Change a shared workflow:** see `platform-demo-workflows/README.md` ("Changing a workflow"): branch, throw-away PRs per language pinning the branch SHA, merge, tag, roll out one PR per repo, publish-test from `main`.
- **Test a gitops change on a branch:** point every `targetRevision:`/`revision:` under `gitops/` at the branch in a throw-away commit, `kubectl apply -f gitops/bootstrap/root.yaml`, revert the commit before the PR.
- **Dependabot:** minor/patch grouped, majors one PR each; toolchain majors were closed with `@dependabot ignore this major version`. Group PRs ignore that command: close them via the API.

## 5. Decisions taken in this session (and why)

| Decision | Why |
|---|---|
| Trunk-based, tags for releases, envs as directories not branches | user's choice; Argo CD + digest promotion by PR |
| Squash-merge only, PR + checks, 0 required approvals | one maintainer; approvals would block self-merge |
| Publish only on `v*` tag / manual dispatch, never on push | user does not want push-triggered pipelines |
| Promote the scanned ghcr digest to Docker Hub, no rebuild | what Trivy scanned is what is signed |
| GitOps config lives in `platform-demo-helm` (`gitops/`, `envs/`) | that repo is already the config repo; one PR per promotion |
| Per-app Applications generated by ApplicationSets (Git files generator over a YAML list) | add app/env = edit a list; RollingSync not used (Beta, forces autosync off) |
| Cilium CNI **and** Cilium Gateway API (Envoy Gateway removed) | user's choice; listeners 8080/8443 in host-network mode because kind has no LoadBalancer |
| Mutual authentication *enforced* (not just WireGuard) | user's choice; Beta feature, authenticates but does not encrypt |
| SPIRE server runs as **root**, persistent 1 Gi PVC | the 1.15 image (uid 1000) breaks the Cilium chart twice (see `kind/README.md`) |
| cosign 3 but **classic** signatures/attestations | Docker Hub has no referrers API; Kyverno/most verifiers expect classic |
| Semgrep via pinned pip install; gradle actions on v5 | `semgrep-action` unmaintained; v6 licence terms |
| `product-catalog-db` = temporary local-test image, no pipeline, RDS on AWS | user's decision; allow-listed (by name) in the Helm cosign gate |
| Argo CD server-side diff on | otherwise Gateway API routes/StatefulSets stay OutOfSync |

## 6. Open items

**Open PRs (all Dependabot, none urgent):** cart-service #4-#9 (NuGet: Google.Protobuf group of 15, coverlet 10, xunit.runner, NetAnalyzers - check against the .NET 10 build before merging), platform-demo-observability #3 (redis 7 -> 8) and #4 (tempo 2.10 -> 3.1, compose images, majors - test the stack).

**Suggested order for the next work:**
1. **Kyverno (the user announced it as the next topic).** Ideas: install via an addon in `envs/<env>/addons.yaml` (chart + values in the repo, own AppProject destination), policies as a chart under `policies/`; enforce Pod Security-equivalents, required labels/resources, **who may set `app.kubernetes.io/name`** (Cilium identity is label-derived: any pod labelled `frontend` *is* the frontend to the network policy), `verifyImages` with the classic cosign signatures and the identity regexp above (note old images were signed by per-repo `ci-main.yaml`), block `:latest`, restrict registries; `PolicyException`s for system namespaces; test with `kyverno test` in the Helm CI; keep RAM in mind (~3 GB free with everything running). The Cilium agent/operator and SPIRE are Beta-grade components: do not let policies break them.
2. **Semantic versioning automation + enforcement** (advice given, not built): conventional-commit PR title check (required, since squash uses the PR title) via a reusable `pr-title.yaml`; release-please per repo opening release PRs (its tags must be created with a GitHub App/fine-grained token or the release must be invoked via `workflow_call`, because tags made with `GITHUB_TOKEN` do not trigger workflows; it normally runs on push to main - run it on schedule/dispatch to keep "nothing on push"); tighten the tag filter from `v*` to `v[0-9]+.[0-9]+.[0-9]+`; `-rc.N` pre-releases that do not move `latest`.
3. **Automate promotion PRs** (Renovate or a small workflow raising "bump digest" PRs after a release) and a second environment (`envs/prod`), possibly with RollingSync across envs.
4. **AWS**: RDS instead of `product-catalog-db` (delete the chart + allow-list entry), cert-manager + ingress/TLS, External Secrets (or Sealed Secrets) for credentials, SPIRE storage on EBS, decide Cilium vs VPC CNI policy support (charts support both via `networkPolicy.type`), EKS cluster registered in Argo CD (cluster generator), Cilium installed by the cluster pipeline.
5. **GitOps for the bootstrap pieces:** adopt Cilium + Gateway API CRDs into Argo CD; NetworkPolicies for `argocd`/`monitoring` namespaces (today only `platform-demo` is policed); WireGuard if the data path must be encrypted; Argo CD SSO instead of the local admin; rotate/delete `argocd-initial-admin-secret`.
6. **Hardening backlog** (unchanged): payment logs full card data (redact before real use), graceful `SIGTERM` in Node/Python, JSON logs for cart, gRPC load balancing (headless Services/mesh), Dockerfile base images pinned by digest, re-review Node `.trivyignore` entries (`CVE-2026-22184`, `CVE-2026-33671`).
7. **Housekeeping:** decide where `README.md`, `OTLP-MIGRATION-PLAN.md`, `HANDOFF*.md` and `shopping-assistant-service/` (GCP-only, no repo) live - none is version-controlled (suggestion: a `platform-demo-docs` repo or `platform-demo-helm/docs/`). Optional: unused `tests/e2e` / `tests/integration` code in cart, frontend, product-catalog (their kind-based CI job was removed).

**Known rough edges:** mutual auth is Beta; a brand-new identity needs 10-15 s before it can talk; `monitoring-local` has no network policies and anonymous Grafana admin (local only); the first `up.sh` after a pull of new images takes longer; chart defaults (`<chart>/values.yaml`) still hold the pre-release digests (envs override them); `sha-8d3a1fe` on Docker Hub (checkout-service) is a leftover from the failed first cosign-3 publish run (image fine, no valid signature) - delete the tag in Docker Hub if it bothers you; `sha-bc9aac8` and `sha-b4633d3` are valid test publishes (`sha-5689788` is part of the real v1.0.0 release).

## 7. Gotchas worth remembering (full list in `kind/README.md` and the workflows README)

- SPIRE 1.15 + Cilium chart: server must run as root; symptoms `Authentication required` drops / `no SPIFFE ID` / `Found 0 entries`. SPIRE 1.15.3 *hangs silently* on a bind error; the 1.12.4 image prints it.
- `actionlint` without shellcheck passes things CI rejects. Download-artifact v8 fails on digest mismatch by default.
- bandit 1.9 flags `random.sample` (B311): recommendation-service uses `random.SystemRandom().sample`.
- Argo CD: `controller.diff.server.side` needs a controller restart to take effect; HPAs without metrics-server make apps Degraded; group Dependabot PRs ignore `@dependabot close`.
- `gh pr edit` fails (Projects classic GraphQL); `kind load docker-image` fails on this machine; musl + VPN DNS search domain needs `ndots: 2`.
- Tags are immutable (ruleset): a wrong tag means the next patch version.

## 8. Side effects on the user's machine

- `pipx install pre-commit`; `~/.cache/pre-commit` (gitleaks built from source with Go 1.26); hooks in `.git/hooks` of all 14 clones; helm repos `argo` and `cilium` added; `kubectl` context for `kind-platform-local` was removed again by `kind delete`.
- Docker images pulled during testing (Cilium, SPIRE, Argo CD, Hubble, Grafana stack, busybox, curl, trivy 0.76, spire-server 1.12.4): `docker image prune -a` reclaims the space.
- Left alone: the user's old kind cluster `platform-demo` (container `platform-demo-control-plane`, exited, cannot start: systemd/cgroup scope error; `kind delete cluster --name platform-demo` removes it) and the stale `kind-crossplane-cluster` context.
- `kind/certs/` (local CA + key, git-ignored) stays; delete it to get a new CA.
- Throw-away tools live in the session scratchpad (`/tmp/claude-.../scratchpad/tools`: actionlint 1.7.12, shellcheck 0.11.0, cosign 2.5.2 and 3.1.3, kubeconform 0.8.0, argocd CLI 3.5.4) and are not needed any more except for manual debugging; versions and checksums were verified against the upstream release checksum files when downloaded.
- The scratch generators were saved to `platform-demo-workflows/tools/`.
