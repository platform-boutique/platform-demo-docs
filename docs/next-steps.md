# Next steps

Priority order; details, decisions and context are in [`HANDOFF.md`](https://github.com/platform-boutique/platform-demo-docs/blob/main/HANDOFF.md). Kyverno (admission control, §13) is done.

1. **Make the developer portal real.** Backstage (catalog from `catalog-info.yaml` per repo, TechDocs, Kubernetes + Argo CD + Kyverno + Grafana plugins, own Hubble and Releases plugins) and single sign-on through **Dex** are built and were tested on the local cluster, but exist only on local branches (nothing pushed). Remaining: create the three new repositories and push, the GitHub OAuth App and a `maintainers` team, catalog discovery from GitHub, the portal's first release, TechDocs publishing in CI; see [`HANDOFF.md`](https://github.com/platform-boutique/platform-demo-docs/blob/main/HANDOFF.md) §0 and the portal's own documentation (`platform-demo-backstage/docs`, rendered in the portal).
2. Infrastructure as code for the platform cluster: **Terraform** (cluster, networking, state) and **Crossplane** (self-service infrastructure), under `infra/`; on AWS: RDS instead of `product-catalog-db`, cert-manager + ingress/TLS, External Secrets, SPIRE storage on EBS, Cilium installed by the cluster pipeline, a registry without anonymous pull limits (ECR pull-through) so image-signature verification can be enforced.
3. Observability depth: synthetic tests, monitors and alerts (SLOs per service, alert rules in Mimir/Grafana), Hubble metrics into Mimir (to retire the Hubble UI/relay pods), policy-report metrics from Kyverno.
4. Semantic-versioning automation and enforcement (§13): conventional-commit PR titles as a required check, release-please opening release PRs, strict `vX.Y.Z` tag filter; automated digest-promotion PRs and a second environment (`envs/prod`).
5. AI tooling on top of the platform.
6. Adopt Cilium (and the Gateway API CRDs) into GitOps; network policies for the `argocd` and `monitoring` namespaces; WireGuard encryption if data-path encryption is required (mutual auth only authenticates).
7. Hardening backlog: redact card data in payment (it logs full card data), graceful `SIGTERM` in Node/Python, JSON logs for cart, headless Services or a mesh for real gRPC balancing, Dockerfile base images pinned by digest, review the Node `.trivyignore` entries; Argo CD SSO instead of the local admin and rotating the initial admin secret.

To continue in a new chat, paste this file, `HANDOFF.md` and the README of what you are working on.

---
