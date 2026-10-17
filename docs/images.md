# Container images

Repository names are `mladenovskistefan/<dir-name>`. Images are built, Trivy-scanned, cosign-signed (keyless) and published to Docker Hub by each service's `release.yaml` (§13): a `v*` tag publishes `:<version>`, `:sha-<short>` and `:latest`. `v1.0.0` was tagged on 2026-10-17, so `:1.0.0` exists for the ten services.

The environments pin **`image.digest`** in `platform-demo-helm/envs/<env>/values/<app>.yaml` (Argo CD deploys exactly these). The `dev` environment runs the **`v1.0.0`** release images:

| Image | Digest deployed by `envs/dev` (`v1.0.0`) |
|---|---|
| frontend | `mladenovskistefan/frontend@sha256:fc355592e4a67d4c10898297f1b5d38422b2859fa5373341fdb2bce857d66715` |
| product-catalog-service | `mladenovskistefan/product-catalog-service@sha256:22c843ec2cc73b20b3a7226877c5b08c141cb0a65e28c02d4257a0db15acb141` |
| checkout-service | `mladenovskistefan/checkout-service@sha256:9d05fb52e2ab1184b45680e1414bddf8cc06e37e55d78cc76b0b652a49879c49` |
| cart-service | `mladenovskistefan/cart-service@sha256:878ad0cf03353ac2f0e8b41edb98443381be561a9a850e7a6e0403e1b0e384ea` |
| currency-service | `mladenovskistefan/currency-service@sha256:d8c4b657e67128c15ae6d2280beef27e81e6374e94a76177a06a48cb0f2911e4` |
| shipping-service | `mladenovskistefan/shipping-service@sha256:cc10d4c55a92c76687b06113caaa07b5c5e6a841d0c5bc046e00cd3ed04d5976` |
| payment-service | `mladenovskistefan/payment-service@sha256:c292f5af4944adbb528e273b9e6822d9890132f56e26301b88bd7c7e27770af4` |
| email-service | `mladenovskistefan/email-service@sha256:e495b8e67397daddf5803037d633e825c298c2a189eb7dabf8bf4afcdb553808` |
| recommendation-service | `mladenovskistefan/recommendation-service@sha256:2ccac2cddeb4ff3759adbfe96e628379bcadb2fed82f201171ae134de286390f` |
| ad-service | `mladenovskistefan/ad-service@sha256:dc106726ade580e7a2ba16d9d39270d89f7afca59809d2667ff1add384a1ce64` |
| product-catalog-db | `mladenovskistefan/product-catalog-db@sha256:c1edd3a805e892df3edee101c046f8a7e45ab38ab670e981cb1a91a4ce3b355b` (chart default; temporary local-testing image) |
| cart-redis | `redis@sha256:8b81dd37ff027bec4e516d41acfbe9fe2460070dc6d4a4570a2ac5b9d59df065` (upstream `redis:7-alpine`, chart default) |

The chart defaults (`<chart>/values.yaml`) still contain the digests of the earlier OTLP builds from the `CI - Main` runs of 2026-10-16/17 (same code, different digests); environments override them. Promotion = a PR that changes the digest in `envs/<env>/values/<app>.yaml`.

- The Helm repo's pull-request pipeline verifies that every `mladenovskistefan/*` image in the rendered charts is pinned by digest **and** carries a cosign signature from one of our pipelines.
- `product-catalog-db` is a **temporary, local-testing image** (Postgres + schema + seed), built by hand, with no pipeline and no signature. When the app is deployed on AWS it is replaced by **RDS** and the image, its chart and its allow-list entry in the Helm CI are deleted.
- `platform-demo-observability/docker-compose.yaml` (the original compose file) still pins the **pre-OTLP** digests, which use the old env contract.

### 4.1 How the images are built (from the Dockerfiles)

All are multi-stage, built with the **service root as build context**.

| Service | Builder base | Runtime base | Build step | Entrypoint | EXPOSE | Docker HEALTHCHECK |
|---|---|---|---|---|---|---|
| frontend | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` | `./frontend` (cwd `/app`) | 8080 | `wget /_healthz` on `${PORT:-8080}` |
| ad-service | `eclipse-temurin:21-jdk-alpine` | `eclipse-temurin:21-jre-alpine` | `./gradlew installDist` | `bin/AdService` | 9555 | none |
| cart-service | `mcr.microsoft.com/dotnet/sdk:10.0` | `mcr.microsoft.com/dotnet/aspnet:10.0` | `dotnet publish -c Release` | `./cartservice` | 7070 | none |
| checkout-service | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` (`GOARCH=amd64`) | `/app/checkout-service` | 5050, 9090 | none |
| currency-service | `node:22.22.2-alpine3.23` | same | `npm ci` + `tsc` + `npm prune --omit=dev` | `node dist/server.js` | 7000 | grpc-js client to `localhost:$PORT` (**needs `PORT` set**) |
| email-service | `python:3.12-alpine` | same | `pip install requirements.in` + `protoc` | `python src/server.py` | 8080 | `channel_ready_future` on `${PORT:-8080}` |
| payment-service | `node:22.22.2-alpine3.23` | same | as currency | `node dist/server.js` | 50051 | grpc-js client to `localhost:$PORT` (**needs `PORT` set**) |
| product-catalog-service | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` (`GOARCH=amd64`) | `/app/product-catalog-service` | 3550, 9090 | none |
| recommendation-service | `python:3.12-alpine` | same | as email | `python src/server.py` | 8080 | `channel_ready_future` on `${PORT:-8080}` |
| shipping-service | `golang:1.26-alpine` | `alpine:3.24` | `go build ./cmd/server` (`GOARCH=amd64`) | `/app/shipping-service` | 50051, 9090 | none |
| product-catalog-db | — | `postgres:17` | copies `dump.sql` to `/docker-entrypoint-initdb.d/001_init.sql` | stock postgres entrypoint | 5432 | none |

Docker `HEALTHCHECK` instructions are **ignored by Kubernetes** — define probes in the chart (§7.3).

Build notes:

- Python services generate their gRPC stubs **inside the Docker build** (stubs are not committed) and install from `requirements.in` (transitive deps float).
- Go services embed generated protobuf code in the repo (`proto/`, `genproto/`).
- `cart-service` now has a root `.dockerignore` (the old one in `src/` had no effect, and a host-built `src/obj` broke the in-image publish with `NETSDK1064`).
- Go services `checkout`, `product-catalog`, `shipping` hard-code `GOARCH=amd64`; the three are the ones that run as root.
- The Node services' runtime stage runs `apk upgrade --no-cache` and removes npm/npx (their bundled `tar`, `pacote`, `sigstore` otherwise trip Trivy). Both keep a `.trivyignore` that should be re-reviewed (`CVE-2026-22184`, `CVE-2026-33671`).
- Base images are pinned by minor tag only (no digests); Dependabot proposes bumps (§13).

---
