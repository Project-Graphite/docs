# Platform registry

The `platform` repository holds the registry. `projects/*.yml` drives project pages and
deployments; the registry sync writes these files from each repository's `.graphite.yml`, and an
entry is edited by hand only for a repository without one, such as `graphite-hub`. The sync skips a
repository whose manifest fails validation or declares a slug registered to another existing
repository, leaves that repository's entries unchanged, and still syncs the rest.
`deploy/<slug>/` says how a project runs in production.

Members are not in the registry. The Hub reads the organisation's members from GitHub, and each
member edits their own page on the Hub.

## Project fields

A repository's `.graphite.yml` lists its entries under `projects`, with every field except `repo`.
The Hub edits `name`, `summary`, `tags`, `status`, `pinned` and `listed` in that file too, through
the platform GitHub App, when a member changes them in the project's editor.

| Field | Required | Meaning |
| :--- | :--- | :--- |
| `slug` | yes | repository name for top-level entries; an epic's child has its own. Also the project's address on the Hub, `project-graphite.com/<slug>`, so it may not equal a member's alias |
| `kind` | yes | `project` or `epic` |
| `parent` | no | slug of a top-level epic in the same repository |
| `name` | yes | display name |
| `summary` | yes | concise present-tense description |
| `tags` | no | up to six lowercase tags |
| `repo` | yes | `project-graphite/<repository>`, set by the sync from the declaring repository |
| `owner` | yes | handle of the member responsible for maintenance; must be an organisation member |
| `authors` | yes | credited GitHub usernames |
| `status` | yes | `live`, `wip`, `archived`, or `source-only` |
| `listed` | no | `false` keeps the entry off the Hub; defaults to `true` |
| `pinned` | no | `true` always shows the entry on the Hub home page, ahead of the others; defaults to `false` |
| `paths` | no | repository folders whose commits belong to this entry's changelog; used by an epic's children |
| `releases` | no | `auto` (default) or `manual`; `manual` records only version tags pushed by people |
| `showcase` | no | problem, approach, outcome, and member contributions |
| `deploy` | yes | `compose`, `pages`, or `none` contract |

## Compose contract

```yaml
deploy:
  kind: compose
  host: example.project-graphite.com
```

The registry only names the host. The services, volumes, health checks, limits and secret names
are in `platform/deploy/<slug>/compose.yaml` and `values`; see the
[platform README](https://github.com/project-graphite/platform#projects).

## Pages contract

```yaml
deploy:
  kind: pages
  host: example.project-graphite.com
  source:
    directory: site
    output: dist
    command: npm run build
```

**Deploy** runs `npm ci` and `command` (default `npm run build`) in `directory` at the commit
`IMAGE_TAG` names, and uploads `directory/output`.

## Validation

```sh
python scripts/validate.py
```

Validation catches schema errors, mismatched filenames and repositories, owners outside the
organisation, duplicate hosts, and invalid deployment sources. It needs `GH_TOKEN` to read the
organisation's members. The Hub validates the same files with the same schema when it reads them,
and skips an entry that fails.
