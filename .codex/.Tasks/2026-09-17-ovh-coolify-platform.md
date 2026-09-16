# OVH and Coolify platform preparation

## 2026-09-17

Replaced the planned GKE deployment path with a cost-bounded OVH VPS-1 and self-hosted Coolify
path. The platform registry now describes Compose deployments, Graphite Message Lab has a
resource-limited production stack for its web, worker, PostgreSQL, and Redis services, and the
project template includes a small production Compose example.

Added an operator runbook covering the order review boundary, Ubuntu preparation, network rules,
Coolify setup, Cloudflare DNS, application configuration, deployment review, backups, capacity,
rollback, and removal. No OVH, Cloudflare, Google Cloud, GitHub, or other external resource was
created, changed, purchased, deployed, or deleted.

Validated the registry and generated catalog, both production Compose definitions, the Message Lab
production images, service health, migrations, worker startup, and all eight application tests.
