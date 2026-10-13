# Platform registry

The `platform` repository holds the registry. `people/*.yml` drives member pages.
`projects/*.yml` drives project pages and deployments; the registry sync writes these files from
each repository's `.graphite.yml`, and an entry is edited by hand only for a repository without
one, such as `graphite-hub`. The sync skips a repository whose manifest fails validation or declares
a slug registered to another existing repository, leaves that repository's entries unchanged, and
still syncs the rest. `tags/*.json` records the image or site build CI last published, and
`deploy/<slug>/values.yaml` says how a project runs in production.

## Project fields

A repository's `.graphite.yml` lists its entries under `projects`, with every field except `repo`.

| Field | Required | Meaning |
| :--- | :--- | :--- |
| `slug` | yes | repository name for top-level entries; an epic's child has its own |
| `kind` | yes | `project` or `epic` |
| `parent` | no | slug of a top-level epic in the same repository |
| `name` | yes | display name |
| `summary` | yes | concise present-tense description |
| `tags` | no | up to six lowercase tags |
| `repo` | yes | `project-graphite/<repository>`, set by the sync from the declaring repository |
| `owner` | yes | member responsible for maintenance |
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
are in `platform/deploy/<slug>/values.yaml`; see the
[platform README](https://github.com/project-graphite/platform#values).

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

**Deploy** runs `npm ci` and `command` (default `npm run build`) in `directory` at the recorded
commit, and uploads `directory/output`.

## Person fields

`people/<handle>.yml` describes one member.

| Field | Required | Meaning |
| :--- | :--- | :--- |
| `handle` | yes | lowercase URL segment: `project-graphite.com/members/<handle>` |
| `github` | yes | GitHub username |
| `profile` | no | `github` (default): the sync fills `name`, `headline` and `links` from the GitHub profile; `manual`: the file is used as written |
| `name` | yes | display name |
| `headline` | no | one line shown under the name |
| `bio` | no | short biography |
| `stack` | no | up to eight `group` labels, each with its `items` |
| `links` | no | up to six `label` and `url` pairs; a URL starts with `https://` or `mailto:` |

## Validation and rendering

```sh
python scripts/validate.py
python scripts/compile.py registry.json
```

Validation catches schema errors, mismatched filenames and repositories, unknown owners, duplicate
hosts, and invalid deployment sources. Secret values are intentionally never generated.
