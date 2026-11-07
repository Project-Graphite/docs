# Troubleshooting

Every entry here is a failure that actually happened while building this, with the cause and the
fix. Most of them present with an error pointing somewhere other than the problem.

---

## A container is repeatedly restarting or reports OOMKilled

Inspect `sudo docker compose -p <slug> logs --tail 100 <service>`, `sudo docker stats --no-stream`
and `free -h`, or the console's Logs page and usage charts. Do not raise a service limit or resize
the VPS without actual usage evidence, the exact recurring price, and approval.

## A container reports an image pull error

Confirm the image name and tag exist in GHCR and that the package is publicly readable. If a package
must remain private, it cannot run here yet: the VPS pulls anonymously. Never put a GitHub token in
the values.

## A project is healthy internally but the hostname fails

Check, in order: the route file `/srv/graphite/edge/routes/<slug>.yml` exists and names the
service that joins the `edge` network as the slug; `sudo docker logs edge-traefik-1` shows no route
or certificate error; the hostname is a DNS-only A record to `135.125.254.59` in Cloudflare,
proxied only for the apex and the console. Rerunning **Deploy** for the project rewrites the route
and the DNS record.

## A host serves "TRAEFIK DEFAULT CERT"

**Cause.** Traefik could not get a Let's Encrypt certificate because the certificate token lacks
Zone → Zone → Read. It needs that and Zone → DNS → Edit on `project-graphite.com`.

**Fix.** Fix the token's permissions or create a new one. Save it as `/edge/CF_DNS_API_TOKEN` on
the console's Secrets page, or edit the edge's env file on the VPS, then run **Deploy** with
`edge`, or on the VPS:

```sh
sudo -u deploy /srv/graphite/bin/graphite-up edge
```

## Deploy fails with "waiting on secrets"

**Cause.** A `${NAME:?}` in the project's compose file has no value in
`/etc/graphite/<slug>.env`. The agent refused before touching anything; the running deployment is
unchanged.

**Fix.** An owner enters the name under `/<slug>/` on the console's Secrets page, which rewrites
the file at once; then rerun **Deploy** for the project. The Secrets page flags the missing names
as **required by compose, missing**.

## A project shows "needs redeploy" and keeps showing it

**Cause.** A secret changed after the project's last rollout; the running containers still have
the old value.

**Fix.** Press Redeploy on the Project page, or rerun **Deploy** for the project. The hint clears
once a rollout finishes after the change.

## The console's Secrets page answers 503

**Cause.** `SECRETS_KEY` is missing from `/etc/graphite/console.env`, or the console was started
with a key that cannot open a stored version (the log names the key id).

**Fix.** Put the key back, or `SECRETS_KEY=<new>,<old>` during a rotation, and redeploy the
console.

## PR fails with "repeats its default; remove the line"

**Cause.** A `values` line equals the `${NAME:-default}` in `shared/` or the project's compose
file. `check-values.py` refuses it, so a default is declared once.

**Fix.** Delete the line.

## A static site deploy says the Pages project is connected to Git

**Cause.** A Git-integrated Cloudflare Pages project already uses the slug. **Deploy** only creates
direct-upload projects, so this happens only if someone recreates one by hand connected to Git.

**Fix.** Delete it in Cloudflare → Workers & Pages and rerun **Deploy** for the site, which creates a
direct-upload project with the same name.

## The Hub's webhook deliveries all get 404

**Symptom.** GitHub's webhook page shows every delivery answered 404, the Hub's
`webhook_deliveries` table is empty, and registry or release changes only appear on the hourly
sync.

**Cause.** The Hub answers 404 to any delivery whose signature does not verify: the secret on the
organisation webhook differs from `/graphite-hub/GITHUB_WEBHOOK_SECRET`, or the latter was changed
without a redeploy.

**Fix.** Set the same secret on both sides, redeploy the Hub, and redeliver one delivery from
GitHub's webhook page; it should answer 202.

## A rollout report records healthStatus 403

**Cause.** `graphite-report` probes the project's host after a rollout. A host behind Cloudflare
Access, such as the console, answers 403 to it; that is Access, not the project. (The Hub's apex
once answered 403 too, until the report script sent a named user agent.)

**Fix.** None needed for Access-protected hosts. For anything else, check the Cloudflare WAF and
the user agent in `host/graphite-report`.

## A workflow run fails with no jobs and no log

**Symptom.** The run is marked failed. Job count is zero. There is no annotation, no log to
download, and the workflow name in the Actions list shows as the file path instead of its `name:`.

That last detail is the tell: **GitHub could not parse or resolve the workflow**, so it never got
as far as reading the name. It is a startup failure, not a runtime one.

Three causes, and `actionlint` catches none of them.

### Cause 1: the caller did not grant the permissions the called workflow needs

A reusable workflow cannot request more than its caller grants. The organisation leaves
`GITHUB_TOKEN` read-only, so `reusable-pr-checks.yml`, which declares `issues: write` and
`pull-requests: write`, fails at startup unless the calling job grants them.

```yaml
  conventions:
    permissions:
      contents: read
      issues: write
      pull-requests: write
    uses: project-graphite/actions/.github/workflows/reusable-pr-checks.yml@main
```

Widening the organisation default to read-write would also fix it, and is the wrong trade: every
workflow everywhere would get write access it does not need.

### Cause 2: a public repository calling a private reusable workflow

GitHub does not allow it. The access policy on a private repository is literally *"Access is
allowed only from private repositories"* — outside collaborators on the calling repository could
otherwise read private workflow logs.

This is why `actions` is public and must stay public. Project repositories are public, so the
workflows they call must be too.

### Cause 3: the called workflow no longer exists

`reusable-sync-registry.yml` was removed from `actions` when the Hub's webhook took over the
dispatch. A repository that still has a `registry.yml` calling it fails this way on every
`.graphite.yml` push. Delete the workflow; the sync runs every hour without it, and sooner when
the Hub's webhook dispatches it.

**How to isolate any of them:** add several small probe workflows in one push, each testing a single
element. A startup failure in one file does not affect the others, so one push gives you several
answers at once.

---

## Org secrets are empty in a private repository

**Symptom.** A secret is empty inside a workflow, and the repository has no repository-level
secrets. The organisation secret exists and looks correctly configured.

**Cause.** On GitHub Free, **organisation secrets are not accessible to private repositories at
all**. No visibility setting changes this; it is a plan limit.

**Fix.** Set them as repository secrets on the private repository:

```sh
gh secret set PLATFORM_APP_CLIENT_ID --repo project-graphite/<repository> --body "<Client ID from the Graphite Deploy app's General settings>"
gh secret set PLATFORM_APP_PRIVATE_KEY --repo project-graphite/<repository> < key.pem
```

Public repositories pick up organisation secrets automatically, which is why every repository
except `traceboard` is public and members cannot create private ones.

---

## Deploy fails with "No registry entry"

**Symptom.** Images build and push fine; the deploy job fails saying `projects/<slug>.yml` does not
exist.

**Cause.** Working as intended. A repository cannot deploy until it is registered.

**Fix.** Add an entry with the repository's slug to its `.graphite.yml` and push it to `main`; the
registry sync, hourly or when the Hub's webhook dispatches it, writes `projects/<slug>.yml`. If the sync skips
the manifest instead, the **Sync registry** run in `platform` lists what to fix. Until then builds
and checks still run and only the record step is blocked, deliberately, so a new repository does
not silently do nothing.

---

## A jq filter fails with "invalid escape sequence"

**Symptom.** `failed to parse jq expression ... invalid escape sequence in string literal`.

**Cause.** Inside a jq string literal a regex dot-escape needs a doubled backslash. A single one is
not a valid string escape, so jq rejects the whole filter.

**Fix.** Use a character class and avoid the escaping question entirely:

```
(package-lock[.]json|yarn[.]lock)$
```

A character class matches a literal dot with no escape at all. The same trick works in JSON Schema
`pattern` values, which have the identical problem, and in any regex that has to survive two levels
of quoting.

---

## Text renders as mojibake

**Symptom.** An em dash or accented character in a registry file comes out as garbage on the site.

**Cause.** On Windows, `pathlib.read_text()` uses the locale encoding — cp1252 — not UTF-8. The
file is UTF-8, so the bytes are misread.

**Fix.** Pass the encoding explicitly, on both reads and writes:

```python
path.read_text(encoding="utf-8")
path.write_text(text, encoding="utf-8")
```

Every script in `platform` does this. It will not reproduce on Linux CI, only locally on Windows,
which makes it easy to ship without noticing.

---

## A YAML value becomes a number

**Symptom.** Validation fails with `name: 2024 is not of type 'string'`.

**Cause.** YAML parses a bare `2024` as an integer.

**Fix.** Quote it: `name: "2024"`. Applies to anything that looks numeric — versions, years, values
with leading zeros.

---

## A shell script fails on Linux with "bad interpreter"

**Cause.** The file was committed with CRLF line endings from a Windows checkout.

**Fix.** Repositories created from the template have `.gitattributes` with `* text=auto eol=lf`.
If a repository was created some other way, add it. Verify what is actually committed:

```sh
git show HEAD:path/to/script.sh | file -
```

`ASCII text` is correct. `with CRLF line terminators` is the problem.

---

## Every URL on a Pages site returns 200, including nonsense

**Symptom.** Any path at all returns HTTP 200 and renders the home page.

**Cause.** Cloudflare Pages falls back to `index.html` for unmatched paths when there is no
`404.html` in the build output. Search engines then index every garbage URL as a real page, and
broken links never surface as broken.

**Fix.** Ship a `404.html` in the build output. Verify with a request to a path that cannot exist;
it should return 404, and the root should still return 200. Test the built output with a plain
static server rather than `vite preview`, which serves `index.html` for any path and so lies.

---

## A static site build fails with "no package.json"

**Cause.** `deploy.source.directory` in the project's `.graphite.yml` does not name the folder that
holds `package.json`.

**Fix.** Set it to that folder. `output` is relative to it, so it stays `dist`, not `site/dist`.

---

## A vanity redirect does nothing

**Cause.** A Bulk Redirect **list** was created but no Bulk Redirect **rule**. The list is data; the
rule activates it.

Also check you are in Account Home rather than the zone — Bulk Redirects are account-level.

---

## Every action on the Hub answers 400 after a deploy

**Cause.** React Router's action handler compares the request's origin with the page's, and
behind Traefik the request arrives as `http`. The Hub sets Express's `trust proxy` for loopback
and private addresses so the forwarded scheme is used; a change that drops it brings this back.

**Fix.** Keep `app.set('trust proxy', 'loopback, uniquelocal')` in `server/app.ts`.

---

## The Hub image starts and never becomes ready

**Cause.** The server imports modules from `app/` at runtime, not only from `build/`. An image that
copies `build/` and `server/` but not `app/` throws on start and never answers `/health/ready`, so
the agent's `--wait` times out and the apex keeps serving the previous container.

**Fix.** The Dockerfile copies `app/`, and the **Image starts** job in CI runs the image against a
Redis until `/health/ready` answers, so this cannot reach `main` again.

---

## An image name is rejected by GHCR

**Cause.** GHCR requires lowercase. A repository name with capitals produces an invalid image name.

**Fix.** Already handled — the build workflow lowercases the repository name. Name repositories in
lowercase anyway.

---

## Docker build cannot find a file in a sibling folder

**Cause.** The build context is the service folder, not the repository root. `server/Dockerfile`
cannot copy from `../shared`.

**Fix.** Add `<service>/.build-contexts` with a `name=./path` line, then `COPY --from=name` in the
Dockerfile, and set the same `additional_contexts` in `compose.yaml`. CI passes the context to the
build and rebuilds the service when that path changes.

---

## Actions minutes are running out

**Cause.** Private repositories share one 2,000 minute monthly pool on the free plan. Public
repositories are unlimited.

**Fix.** Make the repository public. This is the main practical reason project repositories are
public and members cannot create private ones.

---

## Something needs org-level settings and the token refuses

Reading and writing organisation secrets, and the organisation Actions policy, need the `admin:org`
scope.

Think before granting it: **OAuth scopes are account-wide, not per-organisation.** Adding
`admin:org` to a token gives it administrative rights over every organisation you belong to, not
just this one.

If you want a credential confined to this organisation, create a fine-grained personal access token
with `Project-Graphite` as its resource owner. A fine-grained token has exactly one resource owner
and is structurally incapable of reaching anything else.
