# Overview

Graphite Hub publishes member portfolios and project case studies at `project-graphite.com`.
Cloudflare Pages hosts the Hub. Deployable full-stack demonstrations use Docker Compose through
Coolify on one OVH VPS.

```text
member or project pull request
        |
        v
shared checks and image build ---> GHCR SHA and latest images
        |                              |
        v                              v
platform registry and image tags     authenticated deploy webhook
        |                              |
        v                              v
Graphite Hub registry sync       Coolify pulls and health-checks services
                                           |
visitor -> Cloudflare DNS -> Coolify proxy -> project web service
```

Each Compose project provides:

- a versioned production Compose file in its repository
- prebuilt GHCR images, health checks, restart behavior, and explicit resource limits
- named volumes for persistent state
- secret values stored only in Coolify
- one public domain attached only to its web service
- deployment triggered only after the project's tests and image publication succeed

Coolify owns the reverse proxy, HTTPS, deployments, and service logs. Project containers share one
Docker host, so this is operational separation for trusted projects rather than a security boundary
for hostile workloads.

The 4 GB VPS is deliberately a portfolio host, not a high-availability production platform. Static
frontends remain on Cloudflare Pages, images build in GitHub Actions, and the VPS runs only dynamic
services. Sustained swap, OOM events, or less than 10 GB free disk trigger a capacity review before
another project is added.
