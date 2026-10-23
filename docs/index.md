# platform-demo docs

<div class="pd-hero" markdown>
<span class="pd-kicker">docs · master reference</span>

The single source of truth for the platform: what the shop is made of, how it is built, shipped, secured and observed, and what comes next.
</div>

!!! info "Where the platform stands"
    The target is an **AWS EKS** cluster in `us-east-1` for the domain `platform-demo.click`. Terraform ([infra-terraform](https://github.com/platform-boutique/infra-terraform)) builds the AWS side and installs only Argo CD; Argo CD then deploys everything else from [platform-demo-helm](https://github.com/platform-boutique/platform-demo-helm). All of it is merged to `main`, but it has **not yet been deployed or verified on a live cluster**. Pages say so wherever a statement about EKS is still an untested design. The docker compose stack ([local stack](local-stack.md)) is the only environment that has been run end to end under the current setup.

<div class="pd-cards" markdown>
<div><h4><a href="system/">System at a glance</a></h4><p>The ten services, two data stores and the platform around them.</p></div>
<div><h4><a href="architecture/">Architecture</a></h4><p>Where it runs on AWS, and the request flows for browsing and checkout.</p></div>
<div><h4><a href="observability/">Observability</a></h4><p>The OTLP contract every service follows.</p></div>
<div><h4><a href="kubernetes/">Kubernetes</a></h4><p>How the compose layout translates to the cluster.</p></div>
<div><h4><a href="helm/">Helm charts</a></h4><p>The charts, and how Argo CD deploys them layer by layer.</p></div>
<div><h4><a href="cicd/">CI/CD</a></h4><p>Branching, releases, signing, promotion.</p></div>
</div>
