# System at a glance

### The shop

| Item | Value |
|---|---|
| Application services | 10 (`frontend` + 9 backends) |
| Data stores | PostgreSQL 17 (`product-catalog-db`), Redis 7 (`cart-redis`); both run as StatefulSets in the cluster |
| Inter-service protocol | gRPC over plaintext HTTP/2: the services themselves do no TLS and no auth. On the cluster the platform adds the missing layer underneath (SPIFFE mutual authentication and WireGuard, see below) |
| External protocol | HTTP/1.1 from `frontend` only; the browser speaks HTTPS to the load balancer |
| Languages | Go (frontend, checkout, product-catalog, shipping), Java 21 (ad), C#/.NET 10 (cart), TypeScript/Node 22 (currency, payment), Python 3.12 (email, recommendation) |
| Registry | Docker Hub, `mladenovskistefan/<service>`, signed (cosign), pinned by digest in the charts |
| Telemetry | Services **push** traces + metrics + logs over **OTLP/gRPC** to Alloy → Tempo / Mimir / Loki; profiles go to Alloy → Pyroscope. Grafana on top. That stack exists in docker compose only ([local stack](local-stack.md)) |

Services are stateless except the two data stores. Only `frontend` needs to be reachable from outside the cluster.

### The platform around it (AWS EKS)

| Item | Value |
|---|---|
| Where | AWS `us-east-1`, one VPC with two zones; EKS 1.36, two Spot nodes; domain `platform-demo.click` |
| Infrastructure | Terraform in [infra-terraform](https://github.com/platform-boutique/infra-terraform): network, EKS, Cilium, nodes, the `gp3` default StorageClass, ALB, DNS, RDS. Every AWS resource is a Terraform resource; Kubernetes creates none |
| Entry | An ALB (Terraform) terminates TLS and forwards plain HTTP to port 8080 on the nodes, where the Cilium Gateway (Envoy, host network) routes by host name |
| Network | Cilium as the only CNI (overlay, no kube-proxy), SPIRE-based **mutual authentication** required between shop workloads, **WireGuard** between nodes, default-deny network policy |
| GitOps | Terraform installs only Argo CD and one root Application; Argo CD deploys the rest from [platform-demo-helm](https://github.com/platform-boutique/platform-demo-helm) as an app of apps in four layers: bootstrap, security, platform, workloads |
| Admission | Kyverno, **enforcing** for the shop (including signature verification of the images); the platform namespaces are report-only |
| Isolation | gVisor (`runsc`) as an opt-in container runtime; the shop, Backstage and Dex namespaces run in it |
| Secrets | Nothing secret in Git or in Terraform state: External Secrets reads AWS Parameter Store and Secrets Manager; Reloader restarts a workload when its secret changes |
| Databases | Backstage uses a Multi-AZ **RDS** PostgreSQL. The shop's `product-catalog-db` deliberately stays a StatefulSet (the shop is expected to move to its own cluster later) |
| Developer portal | Backstage with GitHub sign-in through Dex; Argo CD and Hubble UI sit behind the same Gateway |
| Monitoring on the cluster | **None for now.** The services still try to export OTLP to `alloy.monitoring`, which does not exist on EKS (log noise only; the services run without it) |
| Status | Merged to `main` in every repository; **not yet deployed**. Nothing in this table has been verified on a live cluster |

Details: [Architecture](architecture.md) for the picture, [Helm charts](helm.md) for the GitOps layers, and the owning repositories' docs (linked from those pages) for the rest.

---
