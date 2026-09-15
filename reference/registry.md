# Registry reference

The registry lives in `platform` and is the single source of truth. Two directories:

- `people/<handle>.yml` — a person and their member portfolio
- `projects/<slug>.yml` — a project, what it is and how it is deployed

Validate before opening a pull request:

```sh
cd platform
pip install pyyaml jsonschema
python scripts/validate.py
```

JSON Schemas live in `platform/schema/`. Point your editor at them for completion and inline
errors — VS Code `settings.json`:

```json
{
  "yaml.schemas": {
    "./schema/project.schema.json": "projects/*.yml",
    "./schema/person.schema.json": "people/*.yml"
  }
}
```

---

## projects/&lt;slug&gt;.yml

| Field | Required | Notes |
| :--- | :--- | :--- |
| `slug` | yes | Lowercase, hyphenated. Must equal the filename and the repository name. |
| `name` | yes | Display name, 2 to 60 characters. **Quote it if numeric** — `name: "2024"`. |
| `summary` | yes | 20 to 240 characters. One or two sentences, present tense. |
| `tags` | no | Up to six lowercase technology tags. |
| `repo` | yes | `project-graphite/<name>`. |
| `owner` | yes | Handle of the member responsible for maintaining it. Must exist in `people/`. |
| `authors` | yes | GitHub usernames of whoever built it. |
| `status` | yes | `live`, `wip`, `archived` or `source-only`. |
| `featured` | no | Sorts the entry to the top of the index. |
| `listed` | no | `false` hides it from the site. For platform pieces only. |
| `deploy.kind` | yes | `compose`, `pages` or `none`. |

**`owner` and `authors` are different things.** `owner` is the member responsible for maintaining
the project in Graphite; `authors` is who actually wrote it. A project appears on a member page when
they maintain it or their GitHub username is in `authors`, so collaborative projects appear on every
registered contributor's portfolio.

An author with a `people/` entry links to their page. Anyone else links to their GitHub profile.

### status

| Value | Card shows | Live link |
| :--- | :--- | :--- |
| `live` | green dot, "live" | yes |
| `wip` | amber dot, "in progress" | no |
| `archived` | grey dot, "archived" | no |
| `source-only` | grey dot, "source only" | no |

Only `live` produces a visit link, so a project cannot advertise a URL that does not resolve.

### deploy.kind: compose

Runs on the shared host. One Docker Compose project, one container per service.

```yaml
deploy:
  kind: compose
  host: atlas.project-graphite.com
  database: postgres
  services:
    frontend: { port: 80, route: /, size: small }
    server:   { port: 3000, route: /api, size: small, env: [OPENAI_API_KEY] }
    worker:   { route: none, size: medium, replicas: 1 }
```

| Service field | Notes |
| :--- | :--- |
| `port` | Container port. Required when `route` is a path. |
| `route` | Path prefix such as `/` or `/api`, or `none` for no public route. |
| `size` | `small` (512 MB, 0.5 CPU) or `medium` (1 GB, 1 CPU). Fixed presets. |
| `replicas` | 1 to 3. Defaults to 1. |
| `env` | Names of variables the service needs. Values live on the host. |
| `writable` | Absolute paths that need to be writable. Containers are read-only apart from `/tmp`. |
| `command` | Overrides the image command. |

Rules validation enforces:

- at least one service must have a path route
- no two services may claim the same route
- every service key must name a folder containing a `Dockerfile`
- `host` must be a first-level subdomain of `project-graphite.com`

`size` is a fixed preset rather than free-form numbers because the host is shared. One project
cannot take the box down.

### deploy.kind: pages

A static build on Cloudflare Pages. No host resources, free, preview deployments per pull request.

```yaml
deploy:
  kind: pages
  host: atlas.project-graphite.com
  source:
    directory: frontend
    output: dist
    command: npm run build
```

`output` is relative to `directory`. Omit `command` for a site with no build step.

### deploy.kind: none

Source published, nothing deployed. Use it for archived work and for anything that is not a web
application — a CLI, a library, a set of algorithm implementations.

```yaml
deploy:
  kind: none
```

---

## people/&lt;handle&gt;.yml

```yaml
handle: amr
github: AmrMsCLL
name: Amr El-Sheraey
headline: Full-stack engineer in Cairo
bio: >
  Two or three sentences. What you work on and what you are interested in.
links:
  - label: github
    url: https://github.com/AmrMsCLL
  - label: linkedin
    url: https://www.linkedin.com/in/amrsherieycll
```

| Field | Required | Notes |
| :--- | :--- | :--- |
| `handle` | yes | Lowercase. Becomes `project-graphite.com/members/<handle>` and must match the filename. |
| `github` | yes | GitHub username. Links `authors` entries to this page. |
| `name` | yes | Display name. |
| `headline` | no | One line, shown under the name. |
| `bio` | no | Up to 600 characters. |
| `links` | no | Up to six labelled links using `https://` or `mailto:`. |

---

## tags/&lt;slug&gt;.json

```json
{
  "frontend": "sha-a1b2c3d",
  "server": "sha-a1b2c3d",
  "worker": "sha-9f8e7d6"
}
```

**Written by CI, never by hand.** Kept separate from the registry entry so a bot deploy never
conflicts with a human edit, and so `git log tags/` reads as a deployment history.

---

## What validation catches that a schema cannot

- filename against `slug` or `handle`
- repository name against `slug`
- an `owner` with no `people/` entry
- two projects claiming the same hostname
- two people claiming the same GitHub account
- two services claiming the same route
- a compose project with no publicly routed service
