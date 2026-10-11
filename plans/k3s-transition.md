# Plan: move to a lightweight k3s

Status: planned, waiting on a larger VPS.

Today `platform` renders each project's values into a Docker Compose project with the `core` chart
and deploys it over SSH. This plan keeps every values file and the secrets repository as they are
and swaps only the runtime: a new major version of the chart renders Kubernetes manifests, and Argo
CD applies them.

## When

When the VPS has at least 4 vCPU and 8 GB. On 2 vCPU and 3.7 GiB, k3s and Argo CD alone take about
2.4 GiB and leave too little for the apps.

| Component | Memory |
| :--- | :--- |
| k3s, with Traefik and ServiceLB turned off | about 1.6 GB |
| Argo CD, slimmed (no Dex, notifications or ApplicationSet) | about 0.6 GiB |
| cloudflared, Sablier or KEDA HTTP add-on, Stakater Reloader | about 0.2 GiB |
| OS and cache | about 0.7 GiB |

On 8 GB that leaves about 4.4 GiB for the apps, which fits today's reservations with room for one
or two more projects. OVHcloud's 2026 VPS-1 (4 vCPU, 8 GB, 75 GB) is the smallest plan that fits;
VPS-2 (6 vCPU, 12 GB) gives headroom. Either is a billing decision.

## What changes

| Piece | Today | After |
| :--- | :--- | :--- |
| `charts/core` | 1.x renders `compose.yaml` and a Traefik route | 2.x renders Deployments, Services, PersistentVolumeClaims, Jobs, CronJobs and an Ingress |
| `deploy/<slug>/values.yaml` | unchanged | unchanged, apart from `core: 2.0.0` |
| `tags/<slug>.json` | read by `deploy.py` | the same file, passed as a values file by Argo CD |
| Deploy | `deploy.py` over SSH | Argo CD pulls `platform`; one Application per project with the chart from tag `core-v2.x` and values from `main` |
| Secrets | **Apply** writes `secrets.env` over SSH | **Apply** writes a Kubernetes Secret `<slug>` in namespace `<slug>` with `kubectl` over SSH, or External Secrets Operator reads AWS SSM |
| Sleeping | Sablier with the Traefik plugin | Sablier's Kubernetes provider, or the KEDA HTTP add-on scaling to zero |
| Edge | cloudflared to Traefik | cloudflared to the k3s ingress |
| Rollback | pin `tag:` in values | the same, or roll back in Argo CD |

How the values map:

| Value | Kubernetes object |
| :--- | :--- |
| a service | Deployment, plus a Service when it has `port` |
| `public: true` | the Ingress rule for the registry host |
| `volumes` | a PersistentVolumeClaim on `local-path` |
| `health` | readiness and liveness probes |
| `dependsOn` | an init container that waits for the dependency |
| `once: true` | a Job (`minio-init`) |
| `resources.memory` / `reserve` | limits and requests |
| `files` | a ConfigMap mounted at the path |
| `secrets` | `envFrom` the project's Secret |

## Steps

1. **Chart 2.0.0.** Add the Kubernetes templates beside the Compose ones, selected by the major
   version, and render all four projects in **PR** with `kubeconform`.
2. **New VPS.** Install k3s with `--disable traefik --disable servicelb`, cloudflared, Argo CD
   (slimmed), Reloader and the sleep controller. Point a second tunnel at it.
3. **Apply.** Teach the secrets **Apply** workflow to write Kubernetes Secrets when the target host
   runs k3s.
4. **Pilot Message Lab.** Back up its database, restore it into the cluster, bump it to
   `core: 2.0.0`, switch DNS to the new tunnel, verify, then remove it from the old VPS.
5. **The rest:** PixelPocket, Graphite Tracker, then Strata.
6. **Retire the old VPS** and the Compose templates, and release chart 3.0.0 without them.

Each project moves by changing one line in its values, and moves back the same way.

## Not in scope

High availability, more than one node, a service mesh and managed databases. One node gives no
redundancy; the move is for the Kubernetes and GitOps workflow, not for uptime.
