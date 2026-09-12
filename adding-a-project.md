# Adding a project, end to end

A worked example for a full project: a front end, an API, a database and a worker. Every step is
here, from creating the repository to the card appearing on the site.

Read the status markers. Steps 1 to 7 are **built and verified**. Step 8, running containers on the
shared host, is **designed but not provisioned** — until a host exists, only `kind: pages` projects
actually serve traffic.

---

## 0. Decide the shape

Answer these before writing anything. Each one becomes a line in the registry entry.

| Question | Example | Becomes |
| :--- | :--- | :--- |
| What is the slug? | `atlas` | repository name, compose project, `/amr/atlas` |
| Who owns it? | `amr` | the namespace it is listed under |
| Who wrote it? | you, a friend, an outside author | the credit line on the card |
| What are the services? | `frontend`, `server`, `worker` | one folder each, one container each |
| Which answer on the web? | `frontend` at `/`, `server` at `/api` | Traefik routes |
| Does it need a database? | yes, Postgres | a database and role provisioned for it |
| What secrets does it need? | `OPENAI_API_KEY` | names in the registry, values on the host |

**Slugs are lowercase and hyphenated.** The slug is the repository name, the subdomain and the URL
segment, so pick something you will not want to rename.

---

## 1. Create the repository

Use the template, not a fork. A fork keeps a link to the parent and complicates everything.

```sh
gh repo create project-graphite/atlas \
  --template project-graphite/project-template \
  --public \
  --description "One sentence on what this is"
git clone https://github.com/project-graphite/atlas
cd atlas
```

**Public.** Members cannot create private repositories in this organisation, and that is
deliberate: on the free plan a private repository cannot read organisation secrets, so its
pipeline would fail in a way that looks like a bug. See
[troubleshooting.md](troubleshooting.md).

What you get: the two CI workflow files, a `compose.yaml`, a `.gitattributes` that forces LF line
endings, an MIT licence, and a placeholder `web/` service that proves the pipeline before you have
written anything.

---

## 2. Lay out the services

**A top-level folder containing a `Dockerfile` is a deployable service.** Nothing declares the
list; CI derives it. Adding a service means adding a folder.

```
atlas/
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── server/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── worker/
│   ├── Dockerfile
│   ├── Makefile
│   └── app/
├── compose.yaml
├── compose.override.yaml
└── README.md
```

Delete the placeholder `web/` folder once you have real services.

**Keep code in a named folder even with only one service.** Image names are
`ghcr.io/project-graphite/<repo>/<service>`, and a service at the repository root has nothing to
call itself.

### The build context is the folder

Each service builds from its own folder. A service cannot reach a sibling during a build.

```dockerfile
# frontend/Dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.29-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

```dockerfile
# server/Dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

```dockerfile
# worker/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 10001
CMD ["celery", "-A", "app.tasks", "worker", "--loglevel=info"]
```

Containers run read-only in production with only `/tmp` writable. If a service needs to write
elsewhere, such as nginx caches, declare those paths in the registry under `writable`.

### Give each service its checks

CI picks the toolchain from what it finds:

| Folder contains | CI runs |
| :--- | :--- |
| `package.json` | `npm ci`, then `lint`, `test` and `build` scripts if they exist |
| `Makefile` | `make install`, `make lint`, `make test` — **all three are required** |
| neither | nothing beyond the image build |

A `Makefile` service owns its own environment. CI does not guess at pip, poetry or uv.

```makefile
# worker/Makefile
install:
	pip install -r requirements.txt -r requirements-dev.txt

lint:
	ruff check .

test:
	pytest -q
```

---

## 3. Make it run locally

`compose.yaml` is what you and everyone else runs. It is not used in production, where the compose
file is generated from the registry, but keeping it accurate is how someone else gets your project
running in one command.

```yaml
name: atlas

services:
  frontend:
    build: ./frontend
    ports:
      - "8080:80"
    depends_on:
      - server

  server:
    build: ./server
    environment:
      DATABASE_URL: postgres://atlas:atlas@db:5432/atlas
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy

  worker:
    build: ./worker
    environment:
      DATABASE_URL: postgres://atlas:atlas@db:5432/atlas
      REDIS_URL: redis://cache:6379
    depends_on:
      - cache

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: atlas
      POSTGRES_PASSWORD: atlas
      POSTGRES_DB: atlas
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U atlas"]
      interval: 5s
      retries: 10
    volumes:
      - db:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine

volumes:
  db:
```

`compose.override.yaml` holds the local-only parts. Docker merges it automatically, so plain
`docker compose up` gives you hot reload and bind mounts while production gets neither.

```yaml
services:
  frontend:
    command: npm run dev -- --host
    volumes:
      - ./frontend/src:/app/src
  server:
    command: npm run start:dev
    volumes:
      - ./server/src:/app/src
```

```sh
docker compose up
```

**The database and cache here are for local development only.** In production Postgres is shared
across projects and provisioned by the host; a `DATABASE_URL` is injected rather than you running
your own.

---

## 4. Write the README

This is what people actually read, and it decides whether the project gets listed.

- one sentence on what it is
- a screenshot or a short GIF
- the live link, once it exists
- the stack
- how to run it locally
- a folder table

---

## 5. Open the first pull request

```sh
git switch -c feat/initial-services
git add -A
git commit -m "feat: add frontend, server and worker"
git push -u origin feat/initial-services
gh pr create --title "feat: add frontend, server and worker" --body "..."
```

**The PR title must read `type(scope): summary`** using one of `feat fix chore refactor docs test
ci perf revert style build`. Merges are squashed and the title becomes the commit message on
`main`, which is why the title is checked and not the branch name.

CI on the pull request:

| Job | What it does |
| :--- | :--- |
| Detect | Lists the services changed in this PR |
| Checks | Lint, test and build each changed service |
| Images | Builds each changed image, **without pushing** |
| Secrets | Scans full history with gitleaks |
| Conventions | PR title format, a size label, path labels |

---

## 6. Register the project

The project does not exist to the organisation until it is in the registry. Until then `main`
builds and pushes images, and the deploy step fails on purpose with a message saying the entry is
missing.

`platform` is private and contributors have read access only. **Maintainers** add the file
directly. **Everyone else** opens an issue in `.github` with the entry filled in, and a maintainer
commits it.

`platform/projects/atlas.yml`:

```yaml
slug: atlas
name: Atlas
summary: >
  One or two sentences, present tense, 20 to 240 characters. What it does, not why
  it is exciting.
tags: [typescript, nestjs, react, postgres]
repo: project-graphite/atlas
owner: amr
authors: [AmrMsCLL]
status: wip
featured: false

deploy:
  kind: compose
  host: atlas.project-graphite.com
  database: postgres
  services:
    frontend:
      port: 80
      route: /
      size: small
    server:
      port: 3000
      route: /api
      size: small
      env: [OPENAI_API_KEY, STRIPE_SECRET_KEY]
    worker:
      route: none
      size: medium
      replicas: 1
```

Every key under `services` must name a folder containing a `Dockerfile`. That is the same rule CI
uses, so the two cannot drift, and validation fails the pull request if they do.

Full field reference: [reference/registry.md](reference/registry.md).

On merge, `platform` CI validates the entry, compiles the registry and commits it to `portfolio`,
which rebuilds. The card appears at `project-graphite.com` and the project page at
`project-graphite.com/amr/atlas`.

---

## 7. Merge to main and watch the build

```
Detect  → frontend, server, worker
Build   → ghcr.io/project-graphite/atlas/frontend:sha-a1b2c3d
          ghcr.io/project-graphite/atlas/server:sha-a1b2c3d
          ghcr.io/project-graphite/atlas/worker:sha-a1b2c3d
Deploy  → writes those tags into platform/tags/atlas.json
```

The images exist, the registry knows about them, and the site lists the project.

---

## 8. Deploy the containers

> **Designed, not provisioned.** There is no host yet. A `kind: compose` project builds, pushes and
> records its tags correctly, but nothing serves `atlas.project-graphite.com` until the host in
> [operations/provisioning-the-host.md](operations/provisioning-the-host.md) exists.

Once it does, the rest is automatic:

1. `platform` CI renders `deploy/atlas/compose.yaml` from the registry entry — images, ports,
   Traefik labels, resource limits, read-only root filesystem, dropped capabilities.
2. Terraform adds the DNS record for `atlas.project-graphite.com` pointing at the tunnel.
3. The reconcile agent notices the commit within a minute and runs `docker compose pull` followed
   by `docker compose up -d`.
4. A database and role are provisioned and `DATABASE_URL` is written into the project secrets file.

**The one manual step is secrets.** The registry names the variables a service needs; the values
never go in git. A maintainer puts them on the host:

```sh
ssh graphite
sudo -e /srv/graphite/secrets/atlas.env
```

The agent refuses to start a project whose declared variables are missing, and names them, rather
than letting it crash-loop.

---

## If the project is static

A front end with no backend skips most of this. Set `kind: pages` and Cloudflare Pages serves it
free, with preview deployments on every pull request.

```yaml
deploy:
  kind: pages
  host: atlas.project-graphite.com
  source:
    directory: frontend
    output: dist
    command: npm run build
```

One manual step, once per project: create the Pages project in Cloudflare and point it at the
repository. See [reference/hosting.md](reference/hosting.md).

---

## Checklist

- [ ] Repository created from the template, public, lowercase hyphenated name
- [ ] Each service in its own top-level folder with a `Dockerfile`
- [ ] Each service has `lint`, `test` and `build` scripts, or a `Makefile`
- [ ] `compose.yaml` brings the whole thing up locally
- [ ] README has a screenshot, the stack and local instructions
- [ ] PR title is `type(scope): summary`
- [ ] Registry entry merged in `platform`
- [ ] Secrets placed on the host, if any
- [ ] Card on the site, project page at `/<owner>/<slug>`, live link works
