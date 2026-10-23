# Repository map

### Workspace layout (local checkout)

Every directory below `apps/`, `deploy/`, `platform/` and `infra/` is its own Git repository (cloned side by side); `docs/` is the repository `platform-demo-docs` (this site). Grouping is by *function*, so new work has an obvious home:

```
platform-demo/
  apps/        what we build: deployable services, one repo each (10 services + product-catalog-db) and a dormant GCP-only assistant without a repo;
               local folder = repo name without the `app-` prefix (apps/frontend is github.com/platform-boutique/app-frontend)
  deploy/      what runs where: platform-demo-helm (charts, envs/<env>/, gitops/, policies/)
  platform/    the machinery around the apps: shared CI/CD workflows, the developer portal (Backstage) and its four plugins,
               observability configs, the organisation profile (org-profile = the `.github` repository)
  infra/       cloud and cluster infrastructure: terraform/ (infra-terraform: VPC, EKS, Cilium, nodes, RDS, ALB, DNS, Argo CD bootstrap)
  docs/        platform-demo-docs: this master reference
```

Convention for what is added later: a new service -> `apps/<name>` (+ a line in `deploy/platform-demo-helm/envs/<env>/workloads.yaml` and its namespace in `envs/<env>/values/cluster-bootstrap.yaml`); anything that provisions clouds or clusters (Terraform, Crossplane compositions) -> `infra/<name>`; developer portal, test/monitoring/alerting code, AI tooling that serves the platform -> `platform/<name>`; environment state -> only `deploy/platform-demo-helm/envs/`. **Naming on GitHub** (an organisation has no folders, so the category is in the name): `app-<service>` for services, `infra-<name>` for infrastructure code (`infra-terraform`; `infra-crossplane` is only an idea), `platform-demo-<name>` for the platform itself (deploy, platform tooling, docs). Every repository also has the custom property `category` (`app`, `deploy`, `platform`, `infra`, `docs`; filter the org list with `props.category:app`) and topics. The organisation front page (repository `platform-boutique/.github`, `profile/README.md`, local folder `platform/org-profile`) lists them by category. Roadmap: the public Project board https://github.com/orgs/platform-boutique/projects/1 (field `Area`).

### Repositories (all in the GitHub organisation `platform-boutique`)

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
| apps | [`app-product-catalog-db`](https://github.com/platform-boutique/app-product-catalog-db) | [README](https://github.com/platform-boutique/app-product-catalog-db#readme) | Postgres image with schema + seed (hand-built, no pipeline, unsigned) |
| apps | [`app-recommendation-service`](https://github.com/platform-boutique/app-recommendation-service) | [README](https://github.com/platform-boutique/app-recommendation-service#readme) | recommendations (Python) |
| apps | [`app-shipping-service`](https://github.com/platform-boutique/app-shipping-service) | [README](https://github.com/platform-boutique/app-shipping-service#readme) | quotes + tracking IDs (Go) |
| deploy | [`platform-demo-helm`](https://github.com/platform-boutique/platform-demo-helm) | [README](https://github.com/platform-boutique/platform-demo-helm#readme) | Helm charts (library, 12 workloads, Gateway, Dex, Backstage, cluster bootstrap, network policies, external-secrets config), `envs/dev/`, `gitops/` (Argo CD app of apps), `policies/` (Kyverno) |
| infra | [`infra-terraform`](https://github.com/platform-boutique/infra-terraform) | [README](https://github.com/platform-boutique/infra-terraform#readme) | Terraform for AWS: stacks `bootstrap`, `foundation`, `platform`; modules network, eks-cluster, cilium (SPIRE, WireGuard), eks-nodes (gVisor), storage-class, argocd-bootstrap, alb, certificate, dns, database (Multi-AZ RDS for Backstage), secrets-access; the Deploy / Destroy / Foundation pipelines. Installs only Argo CD and the root Application, Git does the rest |
| platform | [`platform-demo-workflows`](https://github.com/platform-boutique/platform-demo-workflows) | [README](https://github.com/platform-boutique/platform-demo-workflows#readme) | reusable GitHub Actions workflows: Go, Node, Python, Java, .NET CI; Docker build/scan; Docker promote/sign; secret scan; workflow lint; the Terraform pipeline (`terraform-ci`, `terraform-plan`, `terraform-apply`, v3); generators and protection scripts in `tools/`. Latest release **v3.0.3** (Trivy v0.76.1) |
| platform | [`platform-demo-backstage`](https://github.com/platform-boutique/platform-demo-backstage) | [README](https://github.com/platform-boutique/platform-demo-backstage#readme) | the developer portal (Backstage): catalog, TechDocs, Kubernetes / Argo CD / Grafana plugins, sign-in through Dex; image `mladenovskistefan/platform-demo-backstage` (released with the same pipeline as the services; calls workflows v3.0.3) |
| platform | [`platform-demo-backstage-plugin-hubble`](https://github.com/platform-boutique/platform-demo-backstage-plugin-hubble) | [README](https://github.com/platform-boutique/platform-demo-backstage-plugin-hubble#readme) | portal plugin: Hubble / Cilium flows (Network tab and page) |
| platform | [`platform-demo-backstage-plugin-releases`](https://github.com/platform-boutique/platform-demo-backstage-plugin-releases) | [README](https://github.com/platform-boutique/platform-demo-backstage-plugin-releases#readme) | portal plugin: what Argo CD runs against what the Helm repository declares (Releases tab) |
| platform | [`platform-demo-backstage-plugin-policies`](https://github.com/platform-boutique/platform-demo-backstage-plugin-policies) | [README](https://github.com/platform-boutique/platform-demo-backstage-plugin-policies#readme) | portal plugin: Kyverno findings per workload and the policy catalog (Policies tab and page) |
| platform | [`platform-demo-backstage-plugin-argocd`](https://github.com/platform-boutique/platform-demo-backstage-plugin-argocd) | [README](https://github.com/platform-boutique/platform-demo-backstage-plugin-argocd#readme) | portal plugin: read-only Deployments tab (Argo CD status, history, resources) acting as the signed-in person |
| platform | [`platform-demo-observability`](https://github.com/platform-boutique/platform-demo-observability) | — | docker compose stack: Alloy, Mimir, Loki, Tempo, Pyroscope, Grafana + dashboards; `alloy/config.otlp.alloy` + `docker-compose.otlp.yaml` for the OTLP setup. The only working monitoring stack today |
| platform | [`.github`](https://github.com/platform-boutique/.github) | — | organisation profile (`profile/README.md`), local folder `platform/org-profile` |
| docs | [`platform-demo-docs`](https://github.com/platform-boutique/platform-demo-docs) | this site | master reference, rendered in the portal as TechDocs |

The portal consumes its four plugins as tarballs from its `vendor/` folder until there is a package registry. Every repository has a `catalog-info.yaml` and a TechDocs site (`mkdocs.yml`, `docs/`).

Each `*-service` repo has thin `.github/workflows/ci.yaml` and `release.yaml` callers of the shared workflows ([CI/CD, branching and releases](cicd.md)) and a `tests/` (or `__tests__/`) folder. The ten services still call workflows **v2.0.2**; re-pinning them to v3.0.3 is on the [backlog](next-steps.md) (the docker workflows are identical between the two tags except for the Trivy version). `tests/e2e` and `tests/integration` of cart, frontend and product-catalog target a removed cluster-based job and are not run by CI.

All gRPC APIs share protobuf package `hipstershop` (the Online Boutique heritage). Services keep their own copy of the `.proto` files they use (`proto/` or `src/protos/`), including client-side definitions of the services they call.

---
