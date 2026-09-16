# Overview

Graphite Hub publishes member portfolios and project case studies at `project-graphite.com`.
Cloudflare Pages hosts the Hub. Deployable projects use a shared zonal GKE Standard cluster.

```text
member or project pull request
        |
        v
shared checks and image build ---> GHCR immutable image
        |                              |
        v                              v
platform registry and image tags ---> generated Kubernetes resources
        |                              |
        v                              v
Graphite Hub registry sync       per-project namespace
                                       |
visitor -> Cloudflare Tunnel -> internal gateway -> project Service
```

Each Kubernetes project receives:

- its own namespace, service account, quota, and default resource limits
- default-deny network policies with explicit DNS, internet, intra-project, and gateway access
- a ConfigMap and a separately created Secret
- Deployments and Services for application processes
- optional namespace-local PostgreSQL and Redis StatefulSets with persistent volumes
- an image-specific migration Job

`graphite-system` holds Cloudflare Tunnel and the internal HTTP gateway. There is no public
LoadBalancer or NodePort. One compromised project should not receive credentials or network access
to another project.

The first `e2-small` node is deliberately a lab, not a high-availability production platform. It
fits the system components and one modest full-stack project. More simultaneous projects require a
cost and capacity decision; the namespace and registry structure does not need to change.
