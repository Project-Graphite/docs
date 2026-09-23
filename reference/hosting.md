# Hosting

| Deployment kind | Runtime | Intended use |
| :--- | :--- | :--- |
| `pages` | Cloudflare Pages | Hub and static sites |
| `compose` | Coolify on the shared OVH VPS | APIs, databases, queues, and workers |
| `none` | no runtime | libraries, archived work, and source-only projects |

## Hostnames

```text
project-graphite.com                      Graphite Hub
<subdomain>.project-graphite.com          deployed project
project-graphite.com/members/<handle>     member portfolio
project-graphite.com/projects/<slug>      project case study
project-graphite.com/epics/<slug>         epic
project-graphite.com/epics/<epic>/<slug>  project inside an epic
```

Cloudflare Pages owns the Hub hostname. `compose` subdomains point to the VPS, where Coolify's proxy
maps each hostname to the declared web service and internal port; `pages` subdomains are Cloudflare
Pages custom domains. Database, queue, worker, and application ports are never published directly
on the host.

## Isolation

Each project is a separate Coolify application with its own Compose network, environment values,
containers, and volumes. All containers still share one Docker daemon and Linux kernel. Only
trusted organisation projects belong on this VPS; stronger tenant isolation requires a separate
host or managed runtime.

## Capacity

The initial 4 GB VPS can host Coolify and several small, low-traffic demonstrations when static
frontends stay on Cloudflare Pages and images are built in GitHub Actions. Adding an always-on,
database-heavy project requires reviewing actual memory, swap, disk, and failure impact first.

See [../operations/ovh-coolify.md](../operations/ovh-coolify.md) for the exact user-run setup,
deployment, backup, and cost boundaries.
