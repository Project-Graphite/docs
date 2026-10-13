# Hosting

| Deployment kind | Runtime | Deployed by |
| :--- | :--- | :--- |
| `pages` | Cloudflare Pages, direct upload | platform **Deploy** builds the recorded commit |
| `compose` | Docker Compose on the OVH VPS | platform **Deploy** renders `deploy/<slug>/core.yaml` and `values.yaml` |
| `none` | nothing | — |

## Hostnames

```text
project-graphite.com                      Graphite Hub
<subdomain>.project-graphite.com          deployed project
project-graphite.com/members/<handle>     member portfolio
project-graphite.com/projects/<slug>      project case study
project-graphite.com/epics/<slug>         epic
project-graphite.com/epics/<epic>/<slug>  project inside an epic
deploy.project-graphite.com               the VPS deploy agent
console.project-graphite.com              the console, behind Cloudflare Access
```

**Deploy** owns these DNS records. A `compose` host is a DNS-only A record to the VPS, or a proxied
one when its values set `proxied: true`, as only the console does; a `pages` host is a proxied
CNAME to `<slug>.pages.dev`. VPS hosts use Traefik's Let's Encrypt certificates. Cloudflare's
universal certificate only matters for the proxied hosts, and hostnames stay one level below the
apex so it covers them.

## Routing

Traefik listens on 80 and 443, redirects HTTP to HTTPS, gets each host's certificate from Let's
Encrypt through a Cloudflare DNS challenge, and routes by host to the project's public service on
the `edge` Docker network. No database, queue or app port is published on the host.

## Sleeping

A project with `sleep.after` in its values stops after that long without requests. The next request
shows a waiting page while its containers start, in about half a minute. Bots and monitors get a
plain `200` and do not wake it. `sleep.wake` adds a scheduled wake for projects with background
jobs.

## Isolation and capacity

Each project is its own Compose project with its own network and volumes, sharing one Docker daemon
and kernel. Only trusted organisation projects belong here. Capacity is budgeted in
`platform/deploy/host.yaml`; see [../operations/vps.md](../operations/vps.md#capacity).
