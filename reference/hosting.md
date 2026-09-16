# Hosting

| Deployment kind | Runtime | Intended use |
| :--- | :--- | :--- |
| `pages` | Cloudflare Pages | Hub and static sites |
| `kubernetes` | shared GKE cluster | APIs, databases, queues, and workers |
| `none` | no runtime | libraries, archived work, and source-only projects |

## Hostnames

```text
project-graphite.com                 Graphite Hub
<slug>.project-graphite.com          deployed project
project-graphite.com/members/<name>  member portfolio
project-graphite.com/projects/<slug> project case study
```

Cloudflare Pages owns the Hub hostname. Kubernetes hostnames point at a remotely managed Cloudflare
Tunnel. The tunnel connects to `graphite-gateway.graphite-system.svc.cluster.local:8080`, and the
gateway maps each hostname and path to a ClusterIP Service.

## Isolation

The cluster is shared but projects are not placed in one trust boundary. Each project has a
namespace, quota, non-privileged service account, restricted Pod Security labels, and default-deny
NetworkPolicy. Workload service-account tokens are not mounted. Applications receive only declared
configuration and secret keys.

The design is soft multi-tenancy suitable for trusted organisation members. It is not a sandbox for
hostile workloads. Projects that require strong tenant isolation should use a separate cluster or
runtime.

## Capacity

The initial single `e2-small` node can host one modest full-stack project plus system pods. It is a
cost-bounded deployment test. Adding an always-on database-heavy project requires reviewing node
allocatable memory, namespace requests, persistent disks, projected monthly cost, and failure
impact before changing capacity.

See [../operations/google-cloud-gke.md](../operations/google-cloud-gke.md) for the exact user-run
cluster procedure and cost boundary.
