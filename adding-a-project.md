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

Push to `main`. `platform`'s **Sync registry** runs every hour, and sooner when the Hub's webhook
sees the push and dispatches it; it validates the manifest, writes `projects/<slug>.yml` and merges
it. The project is on the Hub within the hour, and from then on its name, summary, tags, status,
pinned and listed can also be edited on the Hub, which commits them back into `.graphite.yml`.

## 4. Publish images

Each push to `main` publishes `ghcr.io/project-graphite/<repo>/<service>:sha-<short>` and `:latest`
for changed services and writes `IMAGE_TAG=sha-<short>` into `platform/deploy/<slug>/values` once
that folder exists. Make the GHCR packages public once, in the organisation's package settings.

## 5. Write its compose file

Open a pull request in `platform` adding `deploy/<slug>/compose.yaml` and `deploy/<slug>/values`.
The smallest project is:

```yaml
# compose.yaml
services:
  app:
    extends: {file: ../../shared/app.yaml, service: app}
    image: ghcr.io/project-graphite/<repo>/app:${IMAGE_TAG:?}
    environment:
      API_KEY: ${API_KEY:?}
    healthcheck:
      test: [CMD, wget, -qO-, http://127.0.0.1:3000/health]

networks:
  edge: {external: true}
```

```
# values
IMAGE_TAG=sha-1a2b3c4
APP_MEMORY=512M
```

Extend `shared/postgres.yaml` or `shared/redis.yaml` for a database or a cache; label a
PostgreSQL container `graphite.backup.postgres: "true"` and list the volumes to keep under
`graphite.backup.volumes` on the service that owns them, and the nightly backup covers them. Every
default is in `platform/shared/`, and the
[platform README](https://github.com/project-graphite/platform#projects) explains the layout.
`deploy/strata/` is a full example. **PR** renders it with `docker compose config`, refuses a
`${NAME}` without `:?` or `:-`, a `values` line equal to its default, an image without a `sha-`
tag or `${IMAGE_TAG:?}`, a service without a memory limit, and a reservation total over the host
budget.

A `pages` site needs only `deploy/<slug>/values` with `IMAGE_TAG`. Its build comes from
`deploy.source` in `.graphite.yml`.

## 6. Get its secrets added

Ask an organisation owner to enter every `${NAME:?}` in the console under `/<slug>/`, on its
Secrets page, or to grant you that project's secrets there so you can. Until they exist, a deploy
stops with `waiting on secrets: …` and nothing changes on the VPS.

## 7. Deploy and verify

Merging the compose file deploys it: the containers start, the route is published, the hostname
points at the VPS and the rollout appears on the Hub's status page. For a static site, the recorded
commit is built, uploaded to Cloudflare Pages and its hostname pointed at the Pages project. Verify
the site, persistence, background work and logs in the console, then set `status: live` in
`.graphite.yml` and add the outcome to its showcase.

## Checklist

- [ ] local `docker compose up` starts cleanly
- [ ] lint and tests pass, and images publish
- [ ] the compose file renders in **PR** with no errors
- [ ] every long-running service has a health check and a memory limit and reserve
- [ ] exactly one service joins the `edge` network
- [ ] an owner has entered every `${NAME:?}` in the console
- [ ] its data is labelled for backup, and persistence, restart and rollback by editing `IMAGE_TAG` are tested
