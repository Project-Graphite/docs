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

Docker Compose is for local development. Production Kubernetes resources come from the platform
registry, so projects do not carry privileged or divergent production manifests.

## 2. Make the application operable

Each service should have repeatable lint and test commands, a non-root-capable image, deterministic
startup, and a small health endpoint where applicable. Keep state in a declared dependency rather
than the container filesystem. Do not commit credentials or production connection strings.

## 3. Register it

Add `projects/<slug>.yml` to the private `platform` repository. Declare:

- the owner, authors, status, summary, and optional showcase
- `deploy.kind: kubernetes`, `pages`, or `none`
- hostname, services, paths, ports, sizes, commands, and health checks
- non-secret config values and secret names
- optional PostgreSQL, Redis, and migration requirements

Run:

```sh
python scripts/validate.py
python scripts/render_kubernetes.py .rendered
```

Review the generated namespace bootstrap and workloads before merging.

## 4. Publish images

The project workflow publishes
`ghcr.io/project-graphite/<repository>/<service>:sha-<commit>` and records the immutable tag in
`platform/tags/<slug>.json`. The deploy workflow does not have cluster credentials.

## 5. Prepare the namespace

The rendered bootstrap creates the namespace, quota, service account, ConfigMap, and network
policies. Create the declared Kubernetes Secret directly in that namespace. Never put its values in
the registry, repository, command history, issue, or chat.

Applying resources is an external cluster write. Confirm the Kubernetes context, review the exact
diff or server-side dry run, and receive specific approval before applying it.

## 6. Expose and verify

Add the project's public hostname to the existing Cloudflare Tunnel, pointing at the shared
internal gateway. Verify health, persistence, background work, restart behavior, logs, and resource
usage. Add the live link and concrete outcome to its Hub showcase.

## Checklist

- [ ] local Compose environment starts cleanly
- [ ] lint and tests pass
- [ ] images run as non-root under restricted Pod Security
- [ ] registry entry validates and generated YAML is reviewed
- [ ] secret names are declared and values are stored only in Kubernetes
- [ ] requests fit the current node and namespace quota
- [ ] hostname route is configured in Cloudflare Tunnel
- [ ] persistence, worker processing, restart, and rollback are tested
