# GKE commands

```powershell
python scripts/validate.py
python scripts/render_kubernetes.py .rendered
kubectl apply --dry-run=server -f .rendered/cluster/bootstrap.yaml
kubectl apply --dry-run=server -f .rendered/cluster/workloads.yaml
```

Validate the registry, render Kubernetes resources, and ask the API server to validate the shared
system resources without persisting them.
