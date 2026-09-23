# Decisions

## One repository per project

A frontend, API, and worker that form one product stay in one repository. This keeps their shared
contracts and portfolio history together. Deployable service names remain explicit in the platform
registry, and CI publishes a matching immutable image for each one.

## Applications use subdomains; showcases use paths

`atlas.project-graphite.com` is the application. `project-graphite.com/projects/atlas` describes it.
Subdomains isolate browser cookies, storage, and service workers. Hostnames stay one level below the
apex so Cloudflare's standard certificate coverage is sufficient.

## Public project code, private control plane

Project repositories and reusable actions are public. This supports the portfolio, avoids private
Actions-minute limits, and allows public repositories to call shared workflows. `platform` remains
private because it contains deployment topology; it still contains no secrets.

## Human registry data and deployment tags are separate

People edit `.graphite.yml` in their repository and the registry sync writes `projects/<slug>.yml`;
automation edits `tags/<slug>.json`. A deploy cannot conflict with a person editing project
metadata, and tag history remains a readable rollout record.

## Production Compose stays with each project

Each application owns a versioned `compose.production.yaml`. It pulls images built by GitHub
Actions, declares persistent storage and health checks, and caps CPU and memory. The private
platform registry records the Compose source, hostname, public service, port, and secret names.

## Coolify is the shared application control plane

Coolify manages Git sources, deployments, HTTPS routing, environment values, health, and logs on
one OVH VPS. Projects receive separate Compose networks and volumes, but still share a Docker daemon
and kernel. Only trusted organisation projects belong on this host.

## Static work stays off the VPS

Graphite Hub and static project frontends remain on Cloudflare Pages. GitHub Actions builds dynamic
service images. This reserves the 4 GB VPS for APIs, SSR processes, workers, PostgreSQL, and Valkey.

## Every long-running container has a resource ceiling

Production Compose files define CPU and memory limits. Limits prevent one idle demonstration from
consuming the entire portfolio server; actual server memory, swap, disk, and OOM events remain the
capacity signals.

## Application deployment follows successful CI

CI publishes immutable SHA images and moving `main` images only after tests pass. Each automated
application uses a resource-specific Coolify webhook and a deploy-only API token stored as GitHub
Actions secrets. Coolify's direct Auto Deploy remains disabled, preventing a repository event from
racing the image build. Manual Deploy remains the path for first deployment, configuration changes,
and rollback.

## Graphite Hub is prerendered

Every route becomes a real HTML file. Deep links and per-page metadata work on a static host, and
unknown paths return a genuine 404. React is retained for contributor familiarity; prerendering
recovers the important static-site properties.

## Archived projects remain visible

An abandoned project becomes `archived` and loses its live deployment link, but its authorship and
case study remain. This preserves the organisation's history without advertising a dead service.
