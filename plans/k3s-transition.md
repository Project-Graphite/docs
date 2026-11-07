# Plan: move to a lightweight k3s

Status: superseded. The plan assumed the `core` Helm chart would render Kubernetes manifests
beside its Compose output, and that the `secrets` repository's **Apply** workflow would write
Kubernetes Secrets. Both were removed in November 2026: production is plain `compose.yaml` over
`shared/` fragments, and secrets live in the console's store. A move to k3s now starts from those
files, with a converter or a parallel set of manifests, and from the console writing Secrets
instead of env files. Nothing below is scheduled.

## When

When the VPS has at least 4 vCPU and 8 GB. On 2 vCPU and 3.7 GiB, k3s and Argo CD alone take about
2.4 GiB and leave too little for the apps.

| Component | Memory |
| :--- | :--- |
| k3s, with ServiceLB turned off | about 1.6 GB |
| Argo CD, slimmed (no Dex, notifications or ApplicationSet) | about 0.6 GiB |
| Sablier or KEDA HTTP add-on, Stakater Reloader, cert-manager | about 0.2 GiB |
| OS and cache | about 0.7 GiB |

On 8 GB that leaves about 4.4 GiB for the apps, which fits today's reservations with room for one
or two more projects. OVHcloud's 2026 VPS-1 (4 vCPU, 8 GB, 75 GB) is the smallest plan that fits;
VPS-2 (6 vCPU, 12 GB) gives headroom. Either is a billing decision.

## What would change

| Piece | Today | After |
| :--- | :--- | :--- |
| `deploy/<slug>/compose.yaml` and `values` | rendered by `docker compose config`, posted to the agent | the source for Deployments, Services, PersistentVolumeClaims and an Ingress, by a converter kept in `platform` |
| Deploy | `deploy.py` posts to the deploy agent | Argo CD pulls `platform`; one Application per project |
| Secrets | the console writes `/etc/graphite/<slug>.env` | the console writes a Kubernetes Secret `<slug>` in namespace `<slug>` |
| Backups | the shared backup service over the Docker socket | the same script against the cluster's volumes, or Velero |
| Edge | Traefik with Let's Encrypt | the k3s Traefik ingress with cert-manager |
| Rollback | edit `IMAGE_TAG` | the same, or roll back in Argo CD |

## Not in scope

High availability, more than one node, a service mesh and managed databases. One node gives no
redundancy; the move is for the Kubernetes and GitOps workflow, not for uptime.
