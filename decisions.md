# Decisions

## One repository per project

A frontend, API, and worker that form one product stay in one repository. This keeps their shared
contracts and portfolio history together. Deployable service names remain explicit in the platform
registry, and CI publishes a matching immutable image for each one.

## Applications use subdomains; showcases use paths

`atlas.project-graphite.com` is the application. `project-graphite.com/atlas` describes it.
Subdomains isolate browser cookies, storage, and service workers. VPS hosts get Let's Encrypt
certificates from Traefik. Cloudflare's certificate only matters for proxied hosts, the apex and
the console, and hostnames stay one level below the apex so its standard coverage is sufficient.

## Public code, including the control plane

Project repositories and the platform repositories (`platform`, `actions`, `project-template`,
`docs`, `.github`, `graphite-hub`) are public. This supports the portfolio, avoids private
Actions-minute limits, lets organisation secrets reach every repository on the free plan, and makes
`main` rulesets available, so contributors work through reviewed pull requests. `platform` holds
deployment topology but no secrets.

## The registry of record is platform; the Hub reads it and writes back

Superseded by "The Hub's database owns a project's content; the registry owns its infrastructure",
below. Kept as the record of what was decided first.

People edit `.graphite.yml` in their repository and the registry sync writes
`projects/<slug>.yml`, validated with the one JSON schema in `platform`. The Hub reads those
files on a webhook, validates them with the same schema, and commits a registry edit made on the
Hub back into the repository's `.graphite.yml` through the platform GitHub App. One definition of a
project, two readers that cannot disagree. Presentation, such as covers, accents and feature slots,
lives only in the Hub.

## The Hub's database owns a project's content; the registry owns its infrastructure

Reverses the write-back half of "The registry of record is platform; the Hub reads it and writes
back". The registry is still written by the sync from each repository's `.graphite.yml` and still
validated with the one schema. What changed is who owns which field. Infrastructure stays with the
repository: `slug`, `repo`, `parent`, `kind`, `deploy`, `paths` and `releases`; the Hub follows it
on every sync. Content belongs to the Hub's database: `name`, `summary`, `tags`, `status`,
`listed`, `pinned`, `showcase`, `owner` and `authors`, beside the cover, accent, layout and feature
slot it already held. The Hub copies content from the entry once, the first time it sees a slug,
and from then on it is edited on the Hub and never written to a repository.

Nothing outside the Hub read those content fields, so committing them to a repository served no
other reader, and write-back had the Hub commit to, or open pull requests on, every project
repository, which it no longer does.
Three places deciding one field (the entry, a pending edit and a presentation override) gave the
editor two summaries. One rule crosses the boundary: `live` means running at a host, so a project
whose entry stops deploying becomes `archived`. A project the registry drops and a member who
leaves are hidden, not deleted, and return intact.

## Production is a folder in platform, written as plain Compose

Each project's production lives in `platform/deploy/<slug>/`: a `compose.yaml` that extends the
fragments in `shared/`, and a `values` file holding `IMAGE_TAG` and only what differs from a
default. The fragments own the hardening, limits, health checks, PostgreSQL and Redis, so a project
writes only what is particular to it, and a change to how every project runs is one edit to
`shared/`. `docker compose config` is the renderer and the validator; there is no chart, no merge
of two files and no compile step. Project repositories keep their code, Dockerfiles and local
`compose.yaml`.

## Platform deploys everything; Coolify is retired

The platform **Deploy** workflow renders the project and applies it through a signed request to a
small deploy agent on the VPS, and builds the remaining static site into Cloudflare Pages. Git
holds the whole desired state, a deploy is a merge, and a rollback is editing `IMAGE_TAG` back.
Coolify kept a second copy of that state in its own database and UI.

## CI writes the image tag into the values

A project's CI publishes `sha-<short>` images only after its checks pass, then commits
`IMAGE_TAG=sha-<short>` into `platform/deploy/<slug>/values`. The deploy runs from that line, so
it always runs exactly the image CI built, and an automated push may change nothing else.

## Secrets live in the console

Secret values are encrypted in the console's own SQLite, edited on its Secrets page by owners and
by members granted a project, and written to `/etc/graphite/<slug>.env` on the VPS, which
`docker compose` reads at rollout. They never transit CI or the deploy payload: a compose file
names a secret as `${NAME:?}`, and the agent refuses a rollout whose names are missing. The
`secrets` repository, SOPS and its Apply workflow are retired. The console concentrates the risk
where it already is: behind Cloudflare Access, signed in with GitHub, root on the host.

## The apex and the console go through Cloudflare; everything else goes straight to the VPS

Visitors reach Traefik on the VPS directly, with Let's Encrypt certificates, except for the Hub at
the apex and the console. The Hub is proxied for DDoS absorption, the WAF, caching of covers and
share images, and the country header its visitor map needs; the console for Access. CI deploys by
posting signed requests to the deploy agent, so SSH stays open only to the owner's address.

## Graphite Hub is a server

Reverses "Graphite Hub is prerendered" and "Static work stays off the VPS". A static site could
not carry member-edited profiles, a blog written in the browser, or live state. The Hub is one
Node process on the VPS: React Router on Express, SQLite for everything durable, Redis for
everything ephemeral, one replica, no staging. The cost accepted: the apex depends on the VPS, and
a Hub release causes a gap of seconds.

## Every long-running container has a resource ceiling

Every service in a project's compose file sets a memory limit and reserve, and **PR** checks that
the reserves fit the VPS budget. Limits keep one idle demonstration from consuming the server;
actual memory, swap, disk and OOM events remain the capacity signals.

## Backups are shared, verified and importable

One backup service on the VPS covers every project that labels a PostgreSQL container or a volume,
nightly, kept 14 days. Every archive is reopened and checked after it is written, and a failed
check raises a console alert, so a silent bad backup cannot exist. The console's Backups page
downloads, runs and imports archives; the download is the manual off-site path. Scheduled restore
drills were declined in favour of the built-in verification and the import action.

## Archived projects remain visible

An abandoned project becomes `archived` and loses its live deployment link, but its authorship and
case study remain. This preserves the organisation's history without advertising a dead service.
