# Project Graphite recommendations

## Implemented in the OVH and Coolify preparation

- Replaced the GKE-specific registry and renderer with a small Docker Compose contract for Coolify.
- Added a production Message Lab stack that pulls prebuilt GHCR images, persists PostgreSQL and
  Redis, runs migrations, checks health, and caps every service's CPU and memory.
- Kept Graphite Hub and static projects on Cloudflare Pages to preserve VPS capacity.
- Kept secret values outside Git and limited the private registry to their names.
- Added a user-run OVH and Coolify procedure with checkout, security, DNS, deployment, backup,
  rollback, and capacity gates.
- Updated shared workflows and the project template so recording an image release never deploys or
  receives VPS credentials.

## Next, after the first successful lab deployment

1. Prove restoration before treating persistent projects as production. Schedule encrypted database
   exports to separately controlled object storage only after the storage cost and credentials are
   approved.
2. Add a narrowly scoped Coolify deploy webhook only after manual deployment, rollback, secret
   rotation, backup, and restore have each been exercised.
3. Add basic availability and certificate checks from outside the VPS. Alerting should avoid
   expensive log ingestion on the starter budget.
4. Add a Hub deployment-status view sourced from non-sensitive platform metadata: immutable image
   tag, last successful rollout, health state, and project link.
5. Publish an owner-facing runbook for rollback, incident triage, backup restore, and spending
   anomalies before onboarding member-owned stateful services.

## Capacity trigger

Do not resize the VPS pre-emptively. Review capacity when steady memory exceeds 75%, swap use grows,
an OOM event occurs, free disk approaches 10 GB, or another full-stack project must remain online.
Prepare the exact larger VPS, recurring price, tax, backup effect, and downtime before approval.

## Deliberately deferred

- High availability, a second server, load balancer, and managed PostgreSQL exceed the current
  portfolio goal or cost ceiling.
- Kubernetes, service meshes, GitOps suites, and separate hosts per member are unnecessary at this
  scale.
- Preview environments for stateful projects should wait until cleanup automation and cost controls
  are proven.
- Untrusted third-party workloads do not belong on this shared Docker host.
