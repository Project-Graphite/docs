# Platform registry

The private `platform` repository is the source of truth. `people/*.yml` drives member pages;
`projects/*.yml` drives project pages and deployments; `tags/*.json` records immutable image tags.

## Project fields

| Field | Required | Meaning |
| :--- | :--- | :--- |
| `slug` | yes | repository name, namespace, and default hostname prefix |
| `name` | yes | display name |
| `summary` | yes | concise present-tense description |
| `repo` | yes | `project-graphite/<slug>` |
| `owner` | yes | member responsible for maintenance |
| `authors` | yes | credited GitHub usernames |
| `status` | yes | `live`, `wip`, `archived`, or `source-only` |
| `showcase` | no | problem, approach, outcome, and member contributions |
| `deploy` | yes | `kubernetes`, `pages`, or `none` contract |

## Kubernetes contract

```yaml
deploy:
  kind: kubernetes
  host: example.project-graphite.com
  config:
    LOG_LEVEL: info
  secrets: [API_KEY]
  dependencies:
    postgres:
      storage: 5Gi
    redis:
      storage: 1Gi
  migration:
    service: api
    command: [python, manage.py, migrate, --noinput]
  services:
    api:
      port: 8000
      route: /
      size: small
      env: [LOG_LEVEL, API_KEY]
      health:
        readiness: /ready
        liveness: /up
    worker:
      route: none
      size: tiny
      command: [python, worker.py]
      env: [API_KEY]
```

Service names must match GHCR image names. `config` contains non-secret string values. `secrets`
contains names only; values are created directly in the project's Kubernetes Secret. The supported
managed dependencies are namespace-local PostgreSQL and Redis.

Size presets keep reviews predictable:

| Size | CPU request / limit | Memory request / limit |
| :--- | :--- | :--- |
| `tiny` | 25m / 200m | 64 MiB / 128 MiB |
| `small` | 100m / 500m | 192 MiB / 256 MiB |
| `medium` | 250m / 1 CPU | 384 MiB / 768 MiB |

The namespace quota remains the final bound. A `medium` workload or several replicas may not fit on
the starter node even when the schema accepts it.

## Validation and rendering

```sh
python scripts/validate.py
python scripts/render_kubernetes.py .rendered
python scripts/compile.py registry.json
```

Validation catches schema errors, mismatched filenames and repositories, unknown owners, duplicate
hosts, undeclared environment variables, missing migration services, and duplicate routes. The
renderer fails when an expected immutable image tag is missing.

Generated YAML is reviewed and validated but not committed. Secret objects are intentionally never
generated.
