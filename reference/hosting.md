# Hosting

| Deployment kind | Runtime | Deployed by |
| :--- | :--- | :--- |
| `compose` | Docker Compose on the OVH VPS | platform **Deploy** renders `deploy/<slug>/compose.yaml` and posts it to the deploy agent |
| `pages` | Cloudflare Pages, direct upload | platform **Deploy** builds the recorded commit |
| `none` | nothing | — |

## Hostnames

```text
project-graphite.com                      Graphite Hub, proxied through Cloudflare
project-graphite.com/<slug>               project page
project-graphite.com/<epic>/<slug>        project inside an epic
project-graphite.com/<alias>              member page, by the alias they chose
project-graphite.com/members/<handle>     member page, by GitHub login
<subdomain>.project-graphite.com          deployed project
deploy.project-graphite.com               the VPS deploy agent
console.project-graphite.com              the console, proxied, behind Cloudflare Access
```

**Deploy** owns these DNS records. A `compose` host is a DNS-only A record to the VPS, or a proxied
one when its values set `PROXIED=true`, as the Hub and the console do; a `pages` host is a proxied
CNAME to `<slug>.pages.dev`. VPS hosts use Traefik's Let's Encrypt certificates. Cloudflare's
universal certificate only matters for the proxied hosts, and hostnames stay one level below the
apex so it covers them.

A proxied host trusts `CF-Connecting-IP` and `CF-IPCountry` only when the connection comes from
Cloudflare's published ranges, which the Hub refreshes daily.

## Routing

Traefik listens on 80 and 443, redirects HTTP to HTTPS, gets each host's certificate from Let's
Encrypt through a Cloudflare DNS challenge, and routes by host to the service that joins the `edge`
Docker network under the project's name, from one route file per project that the deploy agent
writes. No database, queue or app port is published on the host. No project sleeps: the route the
agent writes has no Sablier middleware, though Sablier still runs in the edge.

## Isolation and capacity

Each project is its own Compose project with its own network and volumes, sharing one Docker daemon
and kernel. Only trusted organisation projects belong here. Capacity is budgeted in
`platform/deploy/host.yaml`; see [../operations/vps.md](../operations/vps.md#capacity).
