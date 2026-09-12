# Hosting

## Two paths

| | `kind: pages` | `kind: compose` |
| :--- | :--- | :--- |
| Runs on | Cloudflare Pages | the shared host |
| Cost | free | shared box |
| Suits | static sites, SPAs | anything with a backend |
| Preview per PR | yes, free | not yet |
| Setup per project | **one manual Cloudflare step** | fully automatic |
| Status | **built and verified** | **designed, not provisioned** |

The automated path is the container one: merging a registry entry creates the DNS record and the
route with no clicking. Pages is the free path with a manual step.

## Domain layout

```
project-graphite.com          the portfolio            Cloudflare Pages
www.project-graphite.com      301 to the apex          Redirect Rule
<slug>.project-graphite.com   a deployed project       Pages, or the host via tunnel
project-graphite.com/<slug>   301 to the subdomain     Bulk Redirect (vanity)
```

Person and project **pages** are routes on the portfolio, not deployments:

```
project-graphite.com/amr          a person and everything they own
project-graphite.com/amr/atlas    a project page: what it is, who built it, what is deployed
```

The running application is always on its own subdomain. `/amr/atlas` is the page *about* Atlas and
links to it.

## Subdomains are one level deep, permanently

Cloudflare Universal SSL on the free plan covers the apex and **first-level subdomains only**.
`atlas.amr.project-graphite.com` would have no valid certificate. Deeper levels need Advanced
Certificate Manager, which is paid.

So subdomains stay flat. If two people ever want the same slug, namespace the slug itself —
`amr-atlas.project-graphite.com` — not the domain.

## Cloudflare Pages setup

Once per `kind: pages` project.

**Workers and Pages → Create → Pages → Connect to Git.** Grant the Cloudflare app access to the
repository; it is scoped per repository, so an earlier grant does not cover a new one.

| Field | Value |
| :--- | :--- |
| Framework preset | React (Vite), or None for a site with no build |
| Root directory | the `source.directory` from the registry entry |
| Build command | the `source.command`, or empty |
| Build output directory | the `source.output`, relative to the root directory |
| `NODE_VERSION` | `22` |

**Root directory is not part of any preset** and is the field that breaks the build when missed.

Then **Custom domains → Set up a custom domain** and enter the hostname from the registry entry.
The DNS record is written for you because the zone is already in the account.

## Vanity redirects

`project-graphite.com/<slug>` redirecting to the subdomain is a **Bulk Redirect**, which is
account-level, not zone-level. Find it under Account Home, not inside the domain.

1. Create a redirect list, add an entry: source `project-graphite.com/atlas`, target
   `https://atlas.project-graphite.com`, status 301, preserve query string, **subpath matching on**,
   preserve path suffix on.
2. Create a **Bulk Redirect Rule** pointing at the list. **The list alone does nothing** — the rule
   is what activates it.

Subpath matching is what makes `project-graphite.com/atlas/settings` carry through rather than 404.

Free plan limits: 10 Single Redirect rules, but **10,000 Bulk Redirect entries**. Use Single
Redirects for the www rule and Bulk Redirects for vanity paths.

**Never use a wildcard for this.** A rule matching `project-graphite.com/*` also matches
`/assets/index-abc123.js` and would redirect the portfolio to nowhere, stripping its own CSS and
JavaScript. There is no way to express "one path segment, no dot" without regex, and regex in Rules
is a Business plan feature.

## The shared host

> **Designed, not provisioned.**

```
cloudflared ──▶ traefik ──▶ project containers
```

Three pieces of infrastructure, then projects. `cloudflared` dials out to Cloudflare, so the box has
no inbound ports open, no exposed IP, and no certificates to renew — Cloudflare terminates TLS and
Traefik speaks plain HTTP internally. Traefik routes by `Host` header, which is why one rule covers
every project and why adding a project needs no infrastructure change.

Details and the provisioning runbook: [../operations/provisioning-the-host.md](../operations/provisioning-the-host.md).
