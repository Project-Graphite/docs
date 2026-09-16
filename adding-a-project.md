# Adding a project

## 1. Create the repository

Use `project-template`. Keep each deployable service in a named folder with a `Dockerfile`:

```text
my-project/
├── frontend/Dockerfile
├── api/Dockerfile
├── worker/Dockerfile
├── compose.yaml
└── compose.override.yaml
```

`compose.yaml` and `compose.override.yaml` are for local development. `compose.production.yaml`
defines the Coolify runtime and must pull prebuilt GHCR images rather than build on the VPS.

## 2. Make the application operable

Each service should have repeatable lint and test commands, a non-root-capable image, deterministic
startup, and a small health endpoint where applicable. Keep state in a declared dependency rather
than the container filesystem. Do not commit credentials or production connection strings.

## 3. Register it

Add `projects/<slug>.yml` to the private `platform` repository. Declare:

- the owner, authors, status, summary, and optional showcase
- `deploy.kind: compose`, `pages`, or `none`
- hostname, production Compose file, public service, and internal port
- secret names without their values

Run:

```sh
python scripts/validate.py
```

Review the registry output before merging.

## 4. Publish images

The project workflow publishes
`ghcr.io/project-graphite/<repository>/<service>:sha-<commit>` and records the immutable tag in
`platform/tags/<slug>.json`. The release-record workflow does not have VPS or Coolify credentials.

## 5. Prepare Coolify

Create a Git-based Docker Compose application in the `Graphite` Coolify project. Select the exact
repository, `main` branch, and production Compose file declared in the registry. Enter secret values
directly in Coolify. Never put them in the registry, repository, command history, issue, or chat.

Selecting Deploy is an external VPS write. Review the parsed services, images, volumes, domains,
health checks, resource limits, and current server capacity before receiving specific approval.

## 6. Expose and verify

Add the project's public hostname in Cloudflare and attach it only to the public service in Coolify.
Verify HTTPS, health, persistence, background work, restart behavior, logs, backups, and resource
usage. Add the live link and concrete outcome to its Hub showcase.

## Checklist

- [ ] local Compose environment starts cleanly
- [ ] lint and tests pass
- [ ] production Compose pulls prebuilt images and publishes no host ports
- [ ] every long-running service has a health check where applicable and a resource limit
- [ ] registry entry validates and matches the production Compose source
- [ ] secret names are declared and values are stored only in Coolify
- [ ] workloads fit the VPS without sustained swap or low disk
- [ ] hostname is configured in Cloudflare and only the web service is public
- [ ] persistence, worker processing, restart, and rollback are tested
