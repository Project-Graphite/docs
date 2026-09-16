# GKE migration preparation

## 2026-09-16

Prepared the organisation repositories for a shared, cost-bounded GKE Standard cluster. The
platform registry now generates isolated per-project Kubernetes resources and the shared internal
gateway and Cloudflare Tunnel deployment. Shared workflows recognize Kubernetes projects, the
template keeps Compose local-only, Graphite Message Lab no longer carries VM deployment files, and
the handbook contains the user-run GCP creation and VM cleanup procedure.

No Google Cloud or Cloudflare resources were created, changed, or deleted.
