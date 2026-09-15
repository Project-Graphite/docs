# Overview

## The repositories

| Repository | Visibility | What it is |
| :--- | :--- | :--- |
| [`.github`](https://github.com/project-graphite/.github) | public | Organisation profile and the contributing guide |
| [`graphite-hub`](https://github.com/project-graphite/graphite-hub) | private | The site at project-graphite.com |
| [`actions`](https://github.com/project-graphite/actions) | public | Reusable CI workflows every repository calls |
| [`platform`](https://github.com/project-graphite/platform) | private | The registry: people, projects, deployment topology |
| [`project-template`](https://github.com/project-graphite/project-template) | private | The skeleton new projects start from |
| `docs` | private | This |
| `<project>` | public | One per project |

`actions` is public deliberately and cannot be made private: **a public repository cannot call a
reusable workflow stored in a private one.** Since project repositories are public, `actions` must
be too. See [decisions.md](decisions.md#why-actions-is-public).

## How a change reaches the site

```
edit platform/projects/<slug>.yml
        │
        ▼
platform CI ── validate ──▶ compile registry.json
        │
        ▼
graphite-deploy app commits to graphite-hub
        │
        ▼
Cloudflare Pages builds and prerenders every route
        │
        ▼
project-graphite.com
```

No manual step anywhere. Verified: a headline edit in `people/amr.yml` reached the live site with
nothing else touched.

## How a code change reaches a deployment

```
push to <project> main
        │
        ▼
detect services ──▶ folders with a Dockerfile that changed
        │
        ▼
build and push to ghcr.io/project-graphite/<repo>/<service>
        │
        ▼
record the image tag in platform/tags/<slug>.json
        │
        ▼
the host reconciles                    ← not provisioned yet
```

The first three steps are built and verified. The last is not; see
[operations/provisioning-the-host.md](operations/provisioning-the-host.md).

## The registry is the source of truth

`platform` holds two directories:

- `people/<handle>.yml` — a person and their portfolio at
  `project-graphite.com/members/<handle>`
- `projects/<slug>.yml` — a project: what it is, who maintains it, who built it and how it is
  deployed

`tags/<slug>.json` records which image tag is currently deployed per service. **It is written by
CI, never by hand.** Keeping it separate from the registry file means a bot deploy never conflicts
with a human edit, and `git log tags/` reads as a deployment history.

Everything the site shows is derived from these files. Nothing about a project is configured in
two places.
