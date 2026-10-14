# Adding a project

## 1. Create the repository

Use `project-template`. Keep each deployable service in a named folder with a `Dockerfile`:

```text
my-project/
├── frontend/Dockerfile
├── api/Dockerfile
├── worker/Dockerfile
└── compose.yaml
```

`compose.yaml` is for local development. Production lives in `platform`.

## 2. Make it operable

Each service needs repeatable lint and test commands, an image that runs without root, a health
endpoint where it serves traffic, and state in a declared volume or dependency. Never commit
credentials or production connection strings.

## 3. Register it

Edit `.graphite.yml`: set `slug` to the repository name, fill in the fields, and set `deploy`:

```yaml
deploy:
  kind: compose                        # compose | pages | none
  host: my-project.project-graphite.com
```

Push to `main`. The **Registry** workflow validates it and `platform` merges `projects/<slug>.yml`.

## 4. Publish images

Each push to `main` publishes `ghcr.io/project-graphite/<repo>/<service>:sha-<short>` and `:latest`
for changed services and records the tag in `platform/tags/<slug>.json`. Make the GHCR packages
public once, in the organisation's package settings.

## 5. Write its values

Open a pull request in `platform` adding `deploy/<slug>/core.yaml`, what runs, and
`deploy/<slug>/values.yaml`, its settings. The smallest project is:

```yaml
# core.yaml
services:
  app:
    image: ghcr.io/project-graphite/<repo>/app
    port: 3000
    health: /health
```

```yaml
# values.yaml
app:
  memory: 256M
  env:
    APP_URL: https://<slug>.project-graphite.com
  secrets: [API_KEY]
```

Add `postgres: {}`, `redis: {}` or `backup: {}` to `core.yaml` for a database, a cache or daily
backups; every service then gets the connection settings. A service can also answer on more hostnames
(`hosts`) and run commands on a schedule (`jobs`). Every option and default is in
`platform/charts/core/values.yaml`, and the
[platform README](https://github.com/project-graphite/platform#projects) explains the layout.
`deploy/strata/` is a full example. **PR** renders it, checks it with Docker Compose, flags names in
`values.yaml` that `core.yaml` does not define and `${NAME}` references missing from `secrets`, and
checks the memory budget.

A `pages` site's values are only `enabled: true` and an optional `tag` pin. Its build comes from
`deploy.source` in `.graphite.yml`.

## 6. Get its secrets added

Ask an organisation owner to add the names listed under each service's `secrets` to
`secrets/<slug>.env`. Until
they exist, a deploy stops with `waiting on secrets: …`.

## 7. Deploy and verify

Merging the values deploys it: the containers start, the route is published and the hostname points
at the VPS. For a static site, the recorded commit is built, uploaded to Cloudflare Pages and its
hostname is pointed at the Pages project. Verify the site, persistence, background work and logs, then set `status: live` in
`.graphite.yml` and add the outcome to its showcase.

## Checklist

- [ ] local `docker compose up` starts cleanly
- [ ] lint and tests pass, and images publish
- [ ] values render in **PR** with no errors
- [ ] every long-running service has a health check and a memory limit and reserve
- [ ] exactly one service is `public`
- [ ] the owner has added every name in `secrets`
- [ ] persistence, restart and rollback by pinning `tag` are tested
