# Platform registry

The private `platform` repository is the source of truth. `people/*.yml` drives member pages;
`projects/*.yml` drives project pages and deployments; `tags/*.json` records immutable image tags.

## Project fields

| Field | Required | Meaning |
| :--- | :--- | :--- |
| `slug` | yes | repository name and default hostname prefix |
| `name` | yes | display name |
| `summary` | yes | concise present-tense description |
| `repo` | yes | `project-graphite/<slug>` |
| `owner` | yes | member responsible for maintenance |
| `authors` | yes | credited GitHub usernames |
| `status` | yes | `live`, `wip`, `archived`, or `source-only` |
| `showcase` | no | problem, approach, outcome, and member contributions |
| `deploy` | yes | `compose`, `pages`, or `none` contract |

## Compose contract

```yaml
deploy:
  kind: compose
  host: example.project-graphite.com
  source:
    file: compose.production.yaml
    service: api
    port: 8000
  secrets: [API_KEY, DATABASE_PASSWORD]
```

`source.file` is the repository-relative production Compose file. `source.service` and
`source.port` identify the single service Coolify exposes for the project hostname. `secrets`
contains names only; values are entered directly in Coolify.

The Compose file is the runtime source of truth. It pulls prebuilt GHCR images, declares required
variables, persists state in named volumes, defines service health checks, publishes no host ports,
and caps CPU and memory for every long-running container.

## Validation and rendering

```sh
python scripts/validate.py
python scripts/compile.py registry.json
```

Validation catches schema errors, mismatched filenames and repositories, unknown owners, duplicate
hosts, and invalid deployment sources. Secret values are intentionally never generated.
