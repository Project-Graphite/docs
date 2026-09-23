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
- Deployed Graphite Message Lab through prebuilt GHCR images and configured an authenticated,
  deploy-only Coolify webhook after the test and image jobs.
- Restricted public host access to SSH, HTTP, and HTTPS; removed direct Coolify dashboard and
  real-time port exposure.
- Enabled Coolify two-factor authentication, closed registration, scheduled instance backup,
  scheduled updates, and non-destructive Docker cleanup.

## Next

1. Prove restoration before treating persistent projects as production. Schedule encrypted database
   exports to separately controlled object storage only after the storage cost and credentials are
   approved.
2. Add basic availability and certificate checks from outside the VPS. Alerting should avoid
   expensive log ingestion on the starter budget.
3. Add a Hub deployment-status view sourced from non-sensitive platform metadata: immutable image
   tag, last successful rollout, health state, and project link.
4. Exercise Message Lab rollback and restore, then record the measured recovery result.
5. Publish an owner-facing incident-triage and spending-anomaly procedure before onboarding
   member-owned stateful services.
6. Review capacity against [operations/ovh-coolify.md §19](operations/ovh-coolify.md#19-capacity-and-billing-boundaries)
   before adding another project.

## Deliberately deferred

- High availability, a second server, load balancer, and managed PostgreSQL exceed the current
  portfolio goal or cost ceiling.
- Kubernetes, service meshes, GitOps suites, and separate hosts per member are unnecessary at this
  scale.
- Preview environments for stateful projects should wait until cleanup automation and cost controls
  are proven.
- Untrusted third-party workloads do not belong on this shared Docker host.
