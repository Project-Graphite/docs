# Adding a project

## 1. Create the repository

Use `project-template`. Keep each deployable service in a named folder with a `Dockerfile`:

```text
my-project/
├── frontend/Dockerfile
├── api/Dockerfile
├── worker/Dockerfile
├── compose.yaml
└── compose.production.yaml
```

`compose.yaml` is for local development. `compose.production.yaml`
defines the Coolify runtime and must pull prebuilt GHCR images rather than build on the VPS.

## 2. Make the application operable

Each service should have repeatable lint and test commands, a non-root-capable image, deterministic
startup, and a small health endpoint where applicable. Keep state in a declared dependency rather
than the container filesystem. Do not commit credentials or production connection strings.

## 3. Register it

Edit `.graphite.yml` at the repository root. The template's copy registers `project-template`
itself, so replace that entry; its comments describe every field. Declare:

- `slug` set to the repository name, with `kind`, `name`, and `summary`
- the owner, authors, status, and optional tags and showcase
- `deploy.kind: compose`, `pages`, or `none`
- for `compose`, the hostname, production Compose file, public service, and internal port
- secret names without their values

Do not add `repo`; the registry sync sets it from the repository name.

Push the change to `main`. `.github/workflows/registry.yml` validates the manifest, then
dispatches `sync-registry` to `platform`, which writes `projects/<slug>.yml` and merges the registry
pull request. An invalid manifest, such as a top-level slug that is not the repository name, fails
the workflow with the problems listed and is not dispatched.

## 4. Publish images

The project workflow publishes
`ghcr.io/project-graphite/<repository>/<service>:sha-<short commit>` and
`ghcr.io/project-graphite/<repository>/<service>:latest`. For a `compose` entry it records the
immutable tag in `platform/tags/<slug>.json`. The release-record workflow does not have VPS or
Coolify credentials.

## 5. Prepare Coolify

Create a Git-based Docker Compose application in the `Graphite` Coolify project. Select the exact
repository, `main` branch, and production Compose file declared in the registry. Enter secret values
directly in Coolify. Never put them in the registry, repository, command history, issue, or chat.

Selecting Deploy is an external VPS write. Review the parsed services, images, volumes, domains,
health checks, resource limits, and current server capacity before receiving specific approval.

## 6. Configure automatic deployment

After the first manual deployment succeeds:

1. keep Coolify's direct Auto Deploy disabled;
2. enable Coolify API access;
3. create a finite-lived API token with only `deploy` permission;
4. copy the application's authenticated resource webhook;
5. store it as `COOLIFY_WEBHOOK` and the token as `COOLIFY_TOKEN` in repository Actions secrets;
6. nothing else: the template's `Deploy` job (`reusable-deploy.yml`) calls the webhook with `POST`
   after tests and image publication succeed, and skips the call while `COOLIFY_WEBHOOK` is unset.

Do not reuse one resource URL across unrelated projects. Never commit the URL or token.

## 7. Expose and verify

Add the project's public hostname in Cloudflare and attach it only to the public service in Coolify.
Verify HTTPS, health, persistence, background work, restart behavior, logs, backups, and resource
usage. Set `status: live` so the Hub links to the hostname, and add the concrete outcome to its
showcase.

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
- [ ] direct Auto Deploy is off and the CI webhook runs only after tests and image publication
