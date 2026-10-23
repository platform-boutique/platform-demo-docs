# Handoff — OTLP migration, Helm charts, and getting the new images built by CI

Written 2026-10-17. Read this, then `README.md` (master reference) and `OTLP-MIGRATION-PLAN.md` (what changed and why).

## Goal

1. All 10 services push traces/metrics/logs over OTLP/gRPC (**code done**, tested locally and in kind).
2. Run each repo's **`CI - Main`** on the branch `feat/otlp-grpc-telemetry` so the pipeline builds and pushes images to Docker Hub (`mladenovskistefan/<svc>`; tags `latest` and `sha-<short>`, cosign-signed, SBOM).
3. Pin the resulting digests in the Helm charts and push `platform-demo-helm`.

User preference: validate through the **remote pipeline**, not long local runs (the user interrupted local `govulncheck` and `npm` runs twice). Make the fix, push, dispatch, read the logs, repeat.

## Access

- `gh` is logged in as `mladenovskistefan111` (token in `~/.config/gh/hosts.yml`, scopes `repo`, `workflow`, `read:org`). `git push` works over HTTPS. Tell the user to revoke the token when finished: https://github.com/settings/tokens
- All 10 service repos already have `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets.
- Dispatch: `gh workflow run ci-main.yaml -R mladenovskistefan111/<repo> --ref feat/otlp-grpc-telemetry`. Logs: `gh run view <id> -R mladenovskistefan111/<repo> --log-failed`.
- Only the **last stage of `CI - Main`** (`Push & Sign Image (Docker Hub)`) publishes images, after lint, secret scan, build, unit tests, SCA, SAST, image build to ghcr.io and a Trivy image scan (HIGH/CRITICAL fails). There is no other image workflow (`CI - Feature` does not push).

## State of the pipelines (first dispatch, branch `feat/otlp-grpc-telemetry`)

| Repo | Run id | Result | Why |
|---|---|---|---|
| cart-service | 38971977801 | **success** | image pushed to Docker Hub (read its digest from the `latest` tag) |
| checkout-service | 38971981192 | failed: SCA | `govulncheck`: Go 1.25.8 stdlib, `x/net`, grpc, otel sdk |
| shipping-service | 38972002129 | failed: SCA | same |
| currency-service | 38971984559 | failed: SCA | `npm audit`: the OpenTelemetry JS 0.5x stack (and protobufjs) |
| payment-service | 38971994385 | failed: SCA | same |
| email-service | 38971987692 | failed: SCA | `pip-audit`: `setuptools 82.0.1` (PYSEC-2026-3447, fix 83.0.0) |
| recommendation-service | 38971998706 | failed: SCA | same |
| ad-service | 38971974491 | failed: Trivy image scan | netty CVEs (e.g. CVE-2026-45416, CVE-2026-50010); `build.gradle` forces netty `4.1.132.Final` |
| frontend | 38971991360 | failed: lint, build, unit tests | `undefined: initTelemetry` — **`cmd/server/otlp.go` was never committed** (see below) |
| product-catalog-service | 38972051878 | failed: Secret Scan | gitleaks finds 3 `generic-api-key` hits in git history in `ci/k8s/product-catalog-db.yaml` (test creds). Dispatch/full-history scan only, push events passed before |

Almost all failures are **advisories published since the last green run in March**, not caused by the telemetry code. Nine of ten need fixes.

## What is committed and pushed

Branch `feat/otlp-grpc-telemetry` (one commit each) in: ad-service, cart-service, checkout-service, currency-service, email-service, frontend, payment-service, product-catalog-service (plus a second commit adding `workflow_dispatch` to its `ci-main.yaml`, because it only triggered on push to `main`), shipping-service, platform-demo-observability. `product-catalog-db` has branch `docs/k8s-readme` (README only, no pipeline). All branched from an up-to-date `main`; nothing was merged to `main`.

## Uncommitted local fixes (in `/home/stefanm/Desktop/platform-demo/<repo>`)

**Go — checkout-service, product-catalog-service, shipping-service, frontend** (done, verified locally: build, vet, tests pass, `govulncheck` clean):
- `go.mod`/`go.sum`: `go 1.26.9` and `go get -u ./...`; `Dockerfile` builder `golang:1.26-alpine` (shipping was pinned `golang:1.25.9-alpine`).
- **frontend only:** `.gitignore` line `server` changed to `/server` (it also ignored `cmd/server/otlp.go`). **`frontend/cmd/server/otlp.go` is untracked and must be committed** (`git add -A`).
- Risk to watch in CI: `golangci-lint` is pinned to `v2.11.4`; it may not accept a Go 1.26 module. Bump the version in the workflow if it complains. Trivy may then flag the `alpine:3.21` runtime base.

**Node — currency-service, payment-service** (edited, **not yet verified**):
- `package.json`/`package-lock.json`: OpenTelemetry JS upgraded to `sdk-node 0.223.0`, `sdk-metrics`/`sdk-trace-base`/`resources` `2.12.0`, `sdk-logs` and the three OTLP gRPC exporters `0.223.0`, `instrumentation-grpc 0.223.0`, `instrumentation-pino 0.69.0`, `semantic-conventions 1.43.0`, `api 1.9.1`.
- `src/telemetry.ts` adapted: `resourceFromAttributes(...)`, `spanProcessors: [...]`, `metricReaders: [...]`, `logRecordProcessors: [new BatchLogRecordProcessor({ exporter })]`.
- To do: `npm ci`, `npm audit --audit-level=high --omit=dev`, `npm run format:check|typecheck|lint|test:ci|build`. Then confirm telemetry still reaches a collector at runtime (the new SDK API is the main risk); an optional local `docker build` + run is enough.

**Not started**
- **email-service, recommendation-service:** add `setuptools>=83.0.0` to `requirements.in` (the Dockerfile installs from it) and regenerate `requirements.txt` (`uv pip compile requirements.in -o requirements.txt`, previously run in a `python:3.13-slim` container). Re-check with `pip-audit -r requirements.txt`.
- **ad-service:** read the full Trivy table in run 38971974491 and raise the forced netty version (and any other flagged library) in `build.gradle`; the CI also runs a Trivy filesystem scan and Semgrep.
- **product-catalog-service:** add a `.gitleaksignore` with the 3 fingerprints from run 38972051878 (`gh run view 38972051878 -R mladenovskistefan111/product-catalog-service --log-failed | grep -i fingerprint`), or an allowlist config.

Then commit, push, dispatch the nine failing pipelines, and iterate on the next failing gate (image CVE scan, E2E). Expect one to three more rounds.

## After the images are in Docker Hub

1. For each service read the digest of `mladenovskistefan/<svc>:latest` (public: `https://hub.docker.com/v2/repositories/mladenovskistefan/<svc>/tags/latest`, field `digest`; or the `sha-<short>` tag matching the commit).
2. Pin `image.digest` in each chart's `values.yaml` (the charts default to tag `1.1.0`, which does **not** exist on Docker Hub). `product-catalog-db` and `cart-redis` already pin digests.
3. The Helm work is committed on a clone, **not pushed**: `/tmp/claude-1000/-home-stefanm-Desktop/ba90c2eb-fc38-4ec2-953a-0ce54bff2f4d/scratchpad/helm-remote/platform-demo-helm`, branch `feat/production-grade-charts` (commit `6da1ad6`, based on remote `main`). The live copy is `/home/stefanm/Desktop/platform-demo/platform-demo-helm` (not a git repo). The chart generator is `.../scratchpad/genchart.py` (regenerating overwrites hand edits). If the scratchpad is gone, clone `https://github.com/mladenovskistefan111/platform-demo-helm`, copy the local folder over it, add the `.gitignore` (ignore each chart's `charts/*.tgz`, keep `Chart.lock`), commit and push.
4. Re-render and lint: `helm lint` on every chart; `helm template` the umbrella with `-f values-dev.yaml`, with defaults plus `--set product-catalog-db.auth.password=x`, and with `-f values-prod.yaml --set product-catalog-db.auth.existingSecret=s`.
5. Update the master `README.md` §4 (image table/digests) and §8.6.

## Things not in any git repo

`/home/stefanm/Desktop/platform-demo/README.md` (master), `OTLP-MIGRATION-PLAN.md`, this file, and `shopping-assistant-service/` (the GCP-only service, no repo). Ask the user where they should live.

## Things to tell the user

- I once ran `git checkout -- .` in `checkout-service` by mistake, discarding its uncommitted work; I re-applied it and re-ran build/vet/tests. Worth a glance at that repo's diff.
- Running `ssh -T git@github.com` added github.com to `~/.ssh/known_hosts`.
- The local machine has Docker images tagged `mladenovskistefan/<svc>:1.1.0` and `local/<svc>:otlp` (never pushed) and Docker volumes `gomodcache`, `gobincache`, `gobincache126`, `gradlecache` that can be removed. The temporary kind cluster was deleted; the user's `kind-platform-demo` / `kind-crossplane-cluster` contexts were not touched.
- The old compose file still pins pre-OTLP digests and keeps working with the old `alloy/config.alloy`; the new setup is `alloy/config.otlp.alloy` + `docker-compose.otlp.yaml`. Dashboards for currency, payment and cart were changed and only match the new images.
- Known unrelated follow-ups: `payment-service` logs full card data, `SIGTERM` handling in Node/Python, JSON logs for cart.
