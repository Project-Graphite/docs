# Decisions

## One repository per project

A frontend, API, and worker that form one product stay in one repository. This keeps their shared
contracts and portfolio history together. Deployable service names remain explicit in the platform
registry, and CI publishes a matching immutable image for each one.

## Applications use subdomains; showcases use paths

`atlas.project-graphite.com` is the application. `project-graphite.com/projects/atlas` describes it.
Subdomains isolate browser cookies, storage, and service workers. VPS hosts get Let's Encrypt
certificates from Traefik. Cloudflare's certificate only matters for proxied hosts, the console and
the Pages sites, and hostnames stay one level below the apex so its standard coverage is sufficient.

## Public code, including the control plane

Project repositories and the platform repositories (`platform`, `actions`, `project-template`,
`docs`, `.github`, `graphite-hub`) are public. This supports the portfolio, avoids private
Actions-minute limits, lets organisation secrets reach every repository on the free plan, and makes
`main` rulesets available, so contributors work through reviewed pull requests. `platform` holds
deployment topology but no secrets.

## Human registry data and deployment tags are separate

People edit `.graphite.yml` in their repository and the registry sync writes `projects/<slug>.yml`;
automation edits `tags/<slug>.json`. A deploy cannot conflict with a person editing project
metadata, and tag history remains a readable rollout record.

## Production is a folder in platform

Each project's production lives in `platform/deploy/<slug>/`, `core.yaml` for what runs and
`values.yaml` for its settings, written by contributors through reviewed pull requests, and the
shared `core` chart renders it. The chart owns the defaults and the common components, PostgreSQL,
Redis and backups, so a project writes only what is particular to it. Every project gets the same
hardening, and a change to how all projects run is one chart release. Project repositories keep
their code, Dockerfiles and local `compose.yaml`.

## Platform deploys everything; Coolify is retired

The platform **Deploy** workflow renders values with Helm into a Docker Compose project and applies
it through a signed request to a small deploy agent on the VPS, and builds static sites into Cloudflare Pages. Git holds the
whole desired state, a deploy is a merge, and a rollback is a pinned tag. Coolify kept a second copy
of that state in its own database and UI. Helm renders Compose today so the same values can move to
a k3s cluster later; see [plans/k3s-transition.md](plans/k3s-transition.md).

## Secrets are owners-only

Secret values are SOPS-encrypted in the private `secrets` repository, which only organisation
owners can see, and its **Apply** workflow writes them to the VPS. Anyone can write values; only an
owner can supply what they reference. Owners are the deliberate bottleneck, the way an operations
team owns a parameter store.

## Traffic goes straight to the VPS

Visitors reach Traefik on the VPS directly, with Let's Encrypt certificates. CI deploys by posting
signed requests to the deploy agent, so SSH stays open only to the owner's address. Of the VPS
hosts, only the console, which is root on the box, sits behind Cloudflare's proxy and Access.

## Idle demonstrations sleep

A project can sleep: Sablier stops it after a period without requests and starts it on the next
one. This keeps occasional demos from holding memory on a small host. Projects with scheduled work
stay awake or set a scheduled wake.

## Static work stays off the VPS

Graphite Hub and static project frontends remain on Cloudflare Pages. GitHub Actions builds dynamic
service images. This reserves the 4 GB VPS for APIs, SSR processes, workers, PostgreSQL, and Redis.

## Every long-running container has a resource ceiling

Every service in a project's values sets a memory limit and reserve, and **PR** checks that the
reserves fit the VPS budget. Limits keep one idle demonstration from consuming the server; actual
memory, swap, disk and OOM events remain the capacity signals.

## Application deployment follows successful CI

A project's CI publishes `sha-<short>` images only after its checks pass, then records the tag in
`platform`. The deploy runs from that record, so it always runs exactly the image CI built, and
values can pin an older tag to roll back.

## Graphite Hub is prerendered

Every route becomes a real HTML file. Deep links and per-page metadata work on a static host, and
unknown paths return a genuine 404. React is retained for contributor familiarity; prerendering
recovers the important static-site properties.

## Archived projects remain visible

An abandoned project becomes `archived` and loses its live deployment link, but its authorship and
case study remain. This preserves the organisation's history without advertising a dead service.
