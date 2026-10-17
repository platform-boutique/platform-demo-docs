# Repository map

### Workspace layout (local checkout)

Every directory below `apps/`, `deploy/`, `platform/` is its own Git repository (cloned side by side); `docs/` is the repository `platform-demo-docs` (this file). Grouping is by *function*, so new work has an obvious home:

```
platform-demo/
  apps/        what we build: deployable services, one repo each (11 + a dormant GCP-only assistant without a repo);
               local folder = repo name without the `app-` prefix (apps/frontend is github.com/platform-boutique/app-frontend)
  deploy/      what runs where: platform-demo-helm (charts, envs/<env>/, gitops/, policies/, kind/)
  platform/    the machinery around the apps: shared CI/CD workflows, observability configs
               (future: backstage/, synthetic tests, monitors/alerts, AI tooling)
  infra/       cloud and cluster infrastructure (future: terraform/ for the platform cluster, crossplane/)
  docs/        platform-demo-docs: this master reference, the handoff, history
  platform-demo.code-workspace    multi-root VS Code workspace for all of the above
```

Convention for what is added later: a new service -> `apps/<name>` (+ a line in `deploy/.../envs/<env>/workloads.yaml`); anything that provisions clouds or clusters (Terraform, Crossplane compositions) -> `infra/<name>`; developer portal, test/monitoring/alerting code, AI tooling that serves the platform -> `platform/<name>`; environment state -> only `deploy/platform-demo-helm/envs/`. **Naming on GitHub** (an organisation has no folders, so the category is in the name): `app-<service>` for services, `infra-<name>` for infrastructure code (planned: `infra-terraform`, `infra-crossplane`), `platform-demo-<name>` for the platform itself (deploy, platform tooling, docs). Every repository also has the custom property `category` (`app`, `deploy`, `platform`, `infra`, `docs`; filter the org list with `props.category:app`) and topics. The organisation front page (repo `platform-boutique/.github`, `profile/README.md`) lists them by category. Roadmap: the public Project board https://github.com/orgs/platform-boutique/projects/1 (field `Area`).

### Repositories (all in the GitHub organisation `platform-boutique`; old `mladenovskistefan111/<repo>` URLs redirect)

| Group | Repository | README | Contents |
|---|---|---|---|
| apps | [`app-frontend`](https://github.com/platform-boutique/app-frontend) | [README](https://github.com/platform-boutique/app-frontend#readme) | HTTP storefront (Go) |
| apps | [`app-ad-service`](https://github.com/platform-boutique/app-ad-service) | [README](https://github.com/platform-boutique/app-ad-service#readme) | ads (Java) |
| apps | [`app-cart-service`](https://github.com/platform-boutique/app-cart-service) | [README](https://github.com/platform-boutique/app-cart-service#readme) | carts on Redis (.NET) |
| apps | [`app-checkout-service`](https://github.com/platform-boutique/app-checkout-service) | [README](https://github.com/platform-boutique/app-checkout-service#readme) | order orchestration (Go) |
| apps | [`app-currency-service`](https://github.com/platform-boutique/app-currency-service) | [README](https://github.com/platform-boutique/app-currency-service#readme) | currency conversion (Node/TS) |
| apps | [`app-email-service`](https://github.com/platform-boutique/app-email-service) | [README](https://github.com/platform-boutique/app-email-service#readme) | email stub (Python) |
| apps | [`app-payment-service`](https://github.com/platform-boutique/app-payment-service) | [README](https://github.com/platform-boutique/app-payment-service#readme) | payment simulation (Node/TS) |
| apps | [`app-product-catalog-service`](https://github.com/platform-boutique/app-product-catalog-service) | [README](https://github.com/platform-boutique/app-product-catalog-service#readme) | catalog API on Postgres (Go) |
| apps | [`app-product-catalog-db`](https://github.com/platform-boutique/app-product-catalog-db) | [README](https://github.com/platform-boutique/app-product-catalog-db#readme) | Postgres image with schema + seed (temporary, local testing; RDS on AWS) |
| apps | [`app-recommendation-service`](https://github.com/platform-boutique/app-recommendation-service) | [README](https://github.com/platform-boutique/app-recommendation-service#readme) | recommendations (Python) |
| apps | [`app-shipping-service`](https://github.com/platform-boutique/app-shipping-service) | [README](https://github.com/platform-boutique/app-shipping-service#readme) | quotes + tracking IDs (Go) |
| apps | `shopping-assistant-service` (local only, no repo) | local README | Gemini/AlloyDB assistant (dormant, GCP-only) |
| deploy | [`platform-demo-helm`](https://github.com/platform-boutique/platform-demo-helm) | [README](https://github.com/platform-boutique/platform-demo-helm#readme) | Helm charts (library + 12 workloads + umbrella + Gateway + local monitoring), `envs/`, `gitops/` (Argo CD), `policies/` (Kyverno), `kind/` runbook |
| platform | [`platform-demo-workflows`](https://github.com/platform-boutique/platform-demo-workflows) | [README](https://github.com/platform-boutique/platform-demo-workflows#readme) | reusable GitHub Actions workflows (Go, Node, Python, Java, .NET CI; Docker build/scan; Docker promote/sign; secret scan; workflow lint) and the generators/protection scripts in `tools/` |
| platform | [`platform-demo-observability`](https://github.com/platform-boutique/platform-demo-observability) | — | compose stack: Alloy, Mimir, Loki, Tempo, Pyroscope, Grafana + dashboards; `alloy/config.otlp.alloy` + `docker-compose.otlp.yaml` for the OTLP setup |
| docs | [`platform-demo-docs`](https://github.com/platform-boutique/platform-demo-docs) | this file | master reference, `HANDOFF.md`, `OTLP-MIGRATION-PLAN.md`, `history/` |

Each `*-service` repo has thin `.github/workflows/ci.yaml` and `release.yaml` callers of the shared workflows (§13) and a `tests/` (or `__tests__/`) folder. `tests/e2e` and `tests/integration` of cart, frontend and product-catalog target the removed kind-based job and are not run by CI.

All gRPC APIs share protobuf package `hipstershop` (the Online Boutique heritage). Services keep their own copy of the `.proto` files they use (`proto/` or `src/protos/`), including client-side definitions of the services they call.

---
