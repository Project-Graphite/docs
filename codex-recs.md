# Project Graphite recommendations

## Implemented in the GKE preparation

- Replaced the production VM and Docker Compose design with registry-generated Kubernetes
  resources. Local Compose remains available for contributors.
- Added a namespace, restricted Pod Security labels, service account without token mounting,
  ResourceQuota, LimitRange, and default-deny NetworkPolicy per project.
- Added shared internal routing through Caddy and Cloudflare Tunnel without a public LoadBalancer or
  NodePort.
- Added immutable GHCR images, health probes, PostgreSQL and Redis StatefulSets, persistent volume
  claims, and image-specific migration Jobs to the platform renderer.
- Kept secret values outside Git and made their absence from generated resources explicit.
- Added a user-run GKE procedure with an initial USD 25 monthly planning ceiling and exact removal
  steps for the abandoned VM path.
- Updated shared workflows, the project template, and Graphite Message Lab so production deployment
  has one contract.

## Next, after the first successful lab deployment

1. Prove restoration before treating persistent projects as production. Schedule encrypted database
   exports to separately controlled object storage only after the storage cost and credentials are
   approved.
2. Add pull-based reconciliation once manual render, dry run, apply, rollback, and secret rotation
   have each been exercised. The reconciler should have namespace-scoped permissions and consume
   reviewed platform commits.
3. Add basic availability and certificate checks from outside the cluster. Alerting should avoid
   expensive log ingestion on the starter budget.
4. Add a Hub deployment-status view sourced from non-sensitive platform metadata: immutable image
   tag, last successful rollout, health state, and project link.
5. Publish an owner-facing runbook for rollback, incident triage, backup restore, and spending
   anomalies before onboarding member-owned stateful services.

## Capacity trigger

Do not add nodes pre-emptively. Review capacity when requested CPU or memory no longer fits, eviction
or OOM events appear, or a second full-stack project must remain online. At that point compare one
larger node with a two-node pool and include the loss of the single-node GKE management-fee credit,
disk, IPv4, and availability tradeoffs in the exact cost proposal.

## Deliberately deferred

- High availability, multi-zone nodes, Cloud NAT, a public ingress load balancer, and managed
  PostgreSQL exceed the current lab goal or cost ceiling.
- Service meshes, Helm operators, GitOps suites, and separate clusters per member are unnecessary at
  this scale.
- Preview environments for stateful projects should wait until cleanup automation and cost controls
  are proven.
- Untrusted third-party workloads do not belong in this soft multi-tenant cluster.
