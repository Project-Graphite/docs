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

The fields fall into two groups, and the Hub treats them differently:

- **Infrastructure** is owned by the repository: `slug`, `repo`, `parent`, `kind`, `deploy`, `paths`
  and `releases`. Change them in `.graphite.yml`; the Hub follows on the next sync and never edits
  them.
- **Content** is owned by the Hub's database: `name`, `summary`, `tags`, `status`, `listed`,
  `pinned`, `showcase`, `owner` and `authors`. The Hub copies them from the entry once, the first
  time it sees the slug. After that they are edited on the Hub and a change in `.graphite.yml` no
  longer reaches the site. The Hub never writes to a repository.

The schema still requires `name`, `summary`, `owner`, `authors` and `status`: they seed the Hub,
and validation checks `owner` against the organisation's members.

| Field | Required | Group | Meaning |
| :--- | :--- | :--- | :--- |
| `slug` | yes | infrastructure | repository name for top-level entries; an epic's child has its own. Also the project's address on the Hub, `project-graphite.com/<slug>`, so it may not equal a member's alias |
| `kind` | yes | infrastructure | `project` or `epic` |
| `parent` | no | infrastructure | slug of a top-level epic in the same repository |
| `name` | yes | content | display name |
| `summary` | yes | content | concise present-tense description |
| `tags` | no | content | up to six lowercase tags |
| `repo` | yes | infrastructure | `project-graphite/<repository>`, set by the sync from the declaring repository |
| `owner` | yes | content | handle of the member responsible for maintenance; must be an organisation member. On the Hub only the organisation owner changes it |
| `authors` | yes | content | credited GitHub usernames. On the Hub only the organisation owner changes them |
| `status` | yes | content | `live`, `wip`, `archived`, or `source-only` |
| `listed` | no | content | `false` keeps the entry off the Hub when it is first seen; defaults to `true` |
| `pinned` | no | content | `true` features the entry on the Hub home page when it is first seen; defaults to `false` |
| `paths` | no | infrastructure | repository folders whose commits belong to this entry's changelog; used by an epic's children |
| `releases` | no | infrastructure | `auto` (default) or `manual`; `manual` records only version tags pushed by people |
| `showcase` | no | content | problem, approach, outcome, and member contributions |
| `deploy` | yes | infrastructure | `compose`, `pages`, or `none` contract |

## What the Hub does with an entry

| The registry | The Hub |
| :--- | :--- |
| lists a slug for the first time | adds the project with the entry's content |
| changes infrastructure on a known slug | follows it; the content stays as edited on the Hub |
| changes content on a known slug | nothing |
| sets `deploy.kind` to `none` on a project whose status on the Hub is `live` | sets the status to `archived`, because `live` means running at a host |
| no longer lists a slug, in a listing where every file is valid | hides the project and stops probing it; its content is kept and returns if the slug does |
| has a file that fails validation | skips that file and removes nothing in that sync |

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
skips an entry that fails, and removes no project in a sync that skipped one.
