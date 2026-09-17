# OVH and Coolify platform

## 2026-09-17 — completed deployment

Purchased the approved USD 5.35 OVH VPS, installed Ubuntu 24.04 and Coolify, added emergency swap,
restricted the OVH edge firewall, removed direct Coolify dashboard port exposure, configured the
dashboard domain and TLS, enabled two-factor authentication, closed registration, and scheduled
instance backup, updates, and Docker cleanup.

Deployed Graphite Message Lab from public GHCR images with persistent PostgreSQL and Redis volumes,
Cloudflare DNS, valid HTTPS, health checks, resource limits, and a Celery worker. Added GitHub
Actions deployment automation using moving `main` images, immutable SHA rollback images, a
resource-specific Coolify webhook, and a deploy-only token. The workflow tests before publishing
images and triggers Coolify with POST only after publication succeeds.

Replaced the preparation-only runbook with the completed purchase-to-redeployment procedure and
updated architecture, decision, onboarding, workflow, recommendations, and repeatable-command
documentation to match the running system.

## 2026-09-17 — preparation

Replaced the planned GKE path with a cost-bounded OVH VPS and self-hosted Coolify design. Converted
the platform registry, Message Lab, project template, and shared workflows from Kubernetes-oriented
deployment to resource-limited production Compose definitions and prebuilt GHCR images.
