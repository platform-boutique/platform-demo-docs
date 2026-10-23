# CI/CD, branching and releases

**Model: trunk-based.** `main` is the trunk; work happens on short-lived `feat/*`, `fix/*`, `docs/*`, `chore/*` branches merged by **squash PR**. There is no production branch: a release is an annotated semver tag `vX.Y.Z` on `main`, and environments are values files in `platform-demo-helm` (promotion = a PR that changes the image digest).

**Pipelines never run on push.** `ci.yaml` runs on `pull_request` (and manual dispatch); `release.yaml` runs on a `v*` tag or manual dispatch (`publish=true`). Both are thin callers of [`platform-demo-workflows`](https://github.com/platform-boutique/platform-demo-workflows#readme), pinned by commit SHA (with the version in a trailing comment) and with explicit per-job `permissions:`. The latest release of the shared workflows is **v3.0.3**: Trivy v0.76.1 in the image, Java and Terraform scans, and the Terraform pipeline v3. The developer portal calls v3.0.3; the ten service repositories still call **v2.0.2** (Trivy 0.70.0) until they are re-pinned (next-steps).

| Stage | What happens |
|---|---|
| language gates | format/lint, gitleaks, build, unit tests (+ E2E and gRPC smoke where enabled), SCA (govulncheck / npm audit / pip-audit / Trivy fs / `dotnet list --vulnerable`), SAST (gosec / Semgrep / bandit) |
| image | build once, stage on ghcr.io, Trivy-scan that exact digest (HIGH/CRITICAL fail) |
| release (tag only) | refuse commits not on `main`; copy the scanned digest to Docker Hub (no rebuild); tags `:X.Y.Z`, `:sha-<short>`, `:latest`; cosign 3.1.3 keyless sign (classic `.sig` format) + self-verify; SBOM attached as a signed `spdxjson` attestation + self-verify |

## The release flow, end to end

```
tag vX.Y.Z on main
  -> release.yaml: gates, build, stage on ghcr.io, Trivy scan of the digest
  -> docker-promote: same digest to Docker Hub, signed keyless with cosign, SBOM attested
  -> PR in platform-demo-helm changing image.tag / image.digest in envs/dev/values/<app>.yaml     (opened BY HAND today)
  -> Helm CI: render, kubeconform, Trivy config, cosign gate, Kyverno policy tests
  -> merge -> Argo CD syncs (Git is polled every 2 minutes)
```

- **A failed release is fixed by a new version.** The tag ruleset `protect-release-tags` forbids deleting or updating `v*` tags, so nobody (admins included) can move a tag to a different commit. If a release fails, publish the next patch version; the failed tag stays as an orphan. This happened to the portal: `v0.1.0` failed its release scan and published nothing, `v0.1.1` is the first portal release.
- **The release workflow must pass the same Trivy ignore file as CI** (`trivyignores:`), otherwise a tag build fails on an accepted finding that the pull request already allowed.
- **Docker Hub push credentials** (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) are secrets of each repository, not of the organisation: a new repository needs them added.
- **Docker Hub has no OCI referrers API.** cosign 3 stores its default bundle format as a referrer, so `cosign sign` succeeds and `cosign verify` then finds nothing. `docker-promote` therefore signs, attests and verifies with `--new-bundle-format=false --use-signing-config=false` (classic `sha256-<digest>.sig` tags), which Kyverno and most verifiers understand. The keyless identity is `https://github.com/platform-boutique/platform-demo-workflows/.github/workflows/docker-promote.yaml@<ref>`; images signed before the move to the organisation carry `mladenovskistefan111` instead, and every verifier (Helm CI, Kyverno) accepts both. The tag listing API may omit `.sig` tags: use `cosign verify` to decide whether a digest is signed.

### Planned: automatic promotion (not built)

Today the promotion pull request is opened by hand. The decided design is a reusable workflow **`gitops-promote`** in `platform-demo-workflows` (to be released as v3.1.0):

- each service's `release.yaml` gets one more job after `docker-promote` that calls it; it opens a PR in `platform-demo-helm` changing `image.tag` and `image.digest` in `envs/dev/values/<app>.yaml`, and enables **auto-merge for dev**, so the merge happens once the Helm CI is green (cosign gate, render, Kyverno policy tests). The portal gets the same job;
- it authenticates as a **GitHub App** (Contents and Pull requests read/write, Metadata read; installed on `platform-demo-helm` only; its id and private key as organisation secrets), not with a personal token. "Allow auto-merge" has to be enabled on the Helm repository;
- later environments (`envs/uat`, `envs/prod`) get the same PR but **reviewed** (CODEOWNERS) instead of auto-merged;
- one PR per repository adds the job (10 services and the portal). Until then, promotion stays manual.

**Helm repository pipeline (PR only):** `helm lint` on every chart, render of the umbrella (dev/default/prod) + gateway **+ every app of every environment exactly as Argo CD renders it** (`.github/scripts/render-envs.py`), kubeconform (strict, CRD schemas) also over the Argo CD manifests, Trivy config scan, cosign gate (every deployed `mladenovskistefan/*` image pinned by digest and signed; `product-catalog-db` allow-listed as unsigned), secret scan, workflow lint, and the **Kyverno policy tests** (unit tests for every policy; every rendered workload/add-on of every environment is evaluated against the policies).

**Terraform pipeline (`infra-terraform`, workflows v3):** pull requests run secret scan, workflow lint, `fmt` + `validate`, TFLint, Trivy config, Checkov, an Infracost estimate and a read-only `plan` as a comment. **Deploy** and **Destroy** are started by hand with a reason: *Plan* (saved encrypted in the state bucket, not as an artifact, because artifacts of a public repository are public), *Review* (the plan must belong to this commit, be younger than 24 hours, be of the right kind and match its checksum; the run page shows the reason and an "at a glance" summary with anything destroyed or replaced first), then *Apply* in a GitHub environment (`production`, or `destruction` for a destroy) that waits for a reviewer and applies exactly the reviewed plan. AWS is reached through GitHub OIDC roles only: the plan role is read-only and cannot read secret values, the apply role can only be assumed from those environments on `main`. No access key is stored anywhere. Details: `docs/pipeline.md` in `infra-terraform`.

## Guardrails

- **Branch protection on `main` in every repository** (22 in the organisation): pull request required, required status checks, linear history, no force-push or deletion, enforced **for admins too**; zero required approvals because there is one maintainer. The required checks differ by repository type (for a Go service: the six language jobs, image build and scan, workflow lint; for Terraform: secret scan, workflow lint, format/validate, TFLint, Trivy, Checkov). Nothing merges red.
- **Squash merges only.**
- **Tag ruleset `protect-release-tags`**: `v*` tags cannot be deleted or updated.
- GitHub secret scanning + push protection, CODEOWNERS, and local `pre-commit` hooks (gitleaks on the staged diff at commit time, on the pushed commits at push time; `--no-verify` bypasses them, push protection and CI do not). Set them up in a fresh clone with `pipx install pre-commit && pre-commit install`.
- **Dependabot** is quiet by design: at most one pull request per repository per month (one multi-ecosystem group, 7-day cooldown), major versions ignored everywhere and Docker base images also ignore minor versions, so a toolchain upgrade is a deliberate decision. The rules live in `.github/dependabot.yml`, not in Dependabot's per-repository memory (a repository transfer wipes that). Security updates are a separate mechanism.
- The organisation is on the **free GitHub plan**: the organisation shares a small number of concurrent CI jobs, so cross-repository changes are rolled out one repository at a time. Nothing in the project may depend on a paid feature (the gitleaks Action needs a paid license in organisations: the pipelines run the open-source gitleaks CLI instead, checksum-pinned).

**Dependency and supply-chain policy:** every action and tool is pinned to the newest release that is at least a week old (a brand-new release is a supply-chain risk), and every `uses:` is pinned to a full commit SHA (checked by `workflow-lint`). `gradle/actions` stays on **v5** because v6 puts its caching component under proprietary terms that nobody accepted. Semgrep runs from a pinned `pip install` (the `semgrep-action` has been unmaintained since 2020). `actionlint` without shellcheck passes things CI rejects: run it with shellcheck installed. Newer bandit flags `random.sample` (B311); recommendation-service uses `random.SystemRandom().sample`.

**Changing a shared workflow:** branch, edit (the language workflows are generated: edit `tools/gen-lang-workflows.py`), open a PR, prove it with throw-away PRs in one repository per language that pin the branch's commit SHA, merge, tag the new version, then roll the new SHA out repository by repository. The publish path can only be exercised from `main`. Breaking input or secret changes get a major version.

### Continuous delivery (Argo CD)

Delivery is pull-based: the pipelines above end at a signed image on Docker Hub; **nothing pushes to the cluster**. `platform-demo-helm` is the desired state (`envs/dev/` + charts); Argo CD renders it and reconciles the cluster as an app of apps (`platform-root` -> layers bootstrap, security, platform, workloads, ordered by sync waves). Two ApplicationSets (workloads, add-ons) generate one Application per environment and app from `envs/*/workloads.yaml` / `addons.yaml`; Argo CD manages its own configuration too. See `gitops/README.md` in the Helm repository and [Kubernetes translation guide](kubernetes.md).

- Terraform installs **only** Argo CD (chart 10.10.0) and the root Application; the version in Git's `application-argocd.yaml` must stay identical to Terraform's, and the AppProject `platform-admin` and `platform-root` exist in both places and must be kept identical.
- Server-side diff is on (`controller.diff.server.side`); without it Gateway API routes and StatefulSets stay OutOfSync. It needs a controller restart to take effect.
- Sync waves gate on the health of child Applications; Applications generated by an ApplicationSet cannot be ordered against each other and rely on retries.
- Automatic sync stays on in dev (a merge is the deploy). Sync, refresh and rollback are deliberately not in the portal: with auto-sync a rollback in the UI is undone, so a rollback is a revert in Git.

### Network security (Cilium)

Cilium is the CNI (VXLAN overlay, kube-proxy replacement, Gateway API on the host network, Hubble) with SPIFFE/SPIRE **mutual authentication** required between shop workloads and **WireGuard** encryption of node-to-node traffic. Workload policies are `CiliumNetworkPolicy` objects generated by `platform-common` (`networkPolicy.type: cilium`): default-deny, explicit callers/dependencies, and `authentication: mode: required` on every workload-to-workload rule; platform namespaces get plain identity-based policy. Mutual authentication was shown to work on a former local cluster (an intruder in another namespace is dropped with `Policy denied`; a pod with an allowed label but no valid identity is dropped with `Authentication required`); **it has not been observed on EKS yet**, and WireGuard on the node AMI is unconfirmed (see [Next steps](next-steps.md)). Limits: mutual authentication is a Beta feature; identity is label-derived, so who may set labels is controlled by Kyverno (below).

### Admission control (Kyverno)

Kyverno 1.19 (chart 3.9.1) is deployed by Argo CD in the security layer, with the policies from `platform-demo-helm/policies/` (CEL policy types, per-environment values). **Enforced for `platform-demo`**, report-only for the platform namespaces (`argocd`, `backstage`, `sso`). On EKS the admission controller runs on the host network (port 9443) because the pod CIDR is not routed in the VPC and the control plane could not reach a webhook in a pod. Full catalog, rollout procedure, exceptions and break-glass: [`policies/README.md`](https://github.com/platform-boutique/platform-demo-helm/blob/main/policies/README.md).

| Concern | What is enforced |
|---|---|
| Network identity | pods in `platform-demo` may only carry one of the 12 workload names in `app.kubernetes.io/name`, must run under the ServiceAccount of the same name, and cannot change the label (Cilium policy and mutual auth identify pods by that label) |
| Supply chain | images only from allowed repositories, pinned by digest, never `:latest`; images of ours verified against the keyless cosign signature (same identity regexp as the Helm CI gate), reading Docker Hub with a read-only token delivered by External Secrets |
| Pod hardening | read-only root filesystem, requests + memory limit, readiness/liveness probes, dedicated ServiceAccount, no API token; mutations add safe defaults (token off, seccomp RuntimeDefault, `backstage.io/kubernetes-id`) |
| Sandbox | namespaces labelled `platform.demo/sandbox=gvisor` (shop, backstage, sso) get `runtimeClassName: gvisor` by mutation; `failurePolicy: Fail`, so an unavailable Kyverno refuses pods rather than starting them unsandboxed |
| Exposure | Services are ClusterIP only, HTTPRoutes only for `*.platform-demo.click`, only Argo CD / admins change network policy, Gateways and routes |
| Operations | `kubectl exec/attach/debug` into workloads only with the break-glass annotation; namespaces keep their Pod Security level; a generated baseline (quota, LimitRange, default-deny CiliumNetworkPolicy) for workload namespaces |

Tested three ways: `kyverno test` (unit cases for every policy), the CI evaluation of the rendered environments, and a live suite (`policies/tests/live/adversarial.sh`, each case asserts which policy denies). The live suite passed on the former local cluster; **it has not been run against EKS** (it is the last step of the first-deploy verification in `gitops/README.md`). Gotchas worth knowing before touching a policy are in [Cross-cutting findings and gotchas](gotchas.md).
