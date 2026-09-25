# Troubleshooting

Every entry here is a failure that actually happened while building this, with the cause and the
fix. Most of them present with an error pointing somewhere other than the problem.

---

## A container is repeatedly restarting or reports OOMKilled

Inspect its Coolify logs and the read-only output of `sudo docker stats --no-stream` and
`free -h`. Stop an unused demonstration before considering a resize. Do not raise a service limit or
resize the VPS without actual usage evidence, the exact recurring price, and approval.

## A container reports an image pull error

Confirm the image name and tag exist in GHCR and that the package is publicly readable. If a package
must remain private, add a narrowly scoped registry credential in Coolify; do not put a GitHub token
in a Compose file.

## A project is healthy internally but the hostname fails

Verify the container health check and Coolify service domain first, then the Coolify proxy, then the
public hostname in Cloudflare. The domain value must include the service's internal port, such as
`:8000`, when the application does not listen on port 80.

## A workflow run fails with no jobs and no log

**Symptom.** The run is marked failed. Job count is zero. There is no annotation, no log to
download, and the workflow name in the Actions list shows as the file path instead of its `name:`.

That last detail is the tell: **GitHub could not parse or resolve the workflow**, so it never got
as far as reading the name. It is a startup failure, not a runtime one.

Two causes, and `actionlint` catches neither.

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

**How to isolate either one:** add several small probe workflows in one push, each testing a single
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
gh secret set PLATFORM_APP_CLIENT_ID --repo project-graphite/platform --body "<Client ID from the Graphite Deploy app's General settings>"
gh secret set PLATFORM_APP_PRIVATE_KEY --repo project-graphite/platform < key.pem
```

Public repositories pick up organisation secrets automatically, which is why project repositories
are public and members cannot create private ones.

---

## Deploy fails with "No registry entry"

**Symptom.** Images build and push fine; the deploy job fails saying `projects/<slug>.yml` does not
exist.

**Cause.** Working as intended. A repository cannot deploy until it is registered.

**Fix.** Add an entry with the repository's slug to its `.graphite.yml` and push it to `main`; the
registry sync writes `projects/<slug>.yml`. If the repository's `Registry` workflow fails instead,
its `Validate .graphite.yml` step lists what to fix. Until then builds and checks still run and only
the record step is blocked, deliberately, so a new repository does not silently do nothing.

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

## Every URL on the site returns 200, including nonsense

**Symptom.** Any path at all returns HTTP 200 and renders the home page.

**Cause.** Cloudflare Pages falls back to `index.html` for unmatched paths when there is no
`404.html` in the build output. Search engines then index every garbage URL as a real page, and
broken links never surface as broken.

**Fix.** Ship a `404.html`. Graphite Hub has one in `site/public/`, so it lands in `dist/`.

Verify with a request to a path that cannot exist; it should return 404, and the root should still
return 200.

---

## A deep link works locally but 404s when deployed, or the reverse

**Cause.** `vite preview` does SPA fallback: it serves `index.html` for any path. Cloudflare Pages
serves directory index files instead. They disagree, so local testing lies.

**Fix.** Test the built output with a plain static server that behaves like Pages:

```sh
cd dist && python -m http.server 4173
```

Graphite Hub does not rely on SPA fallback at all. Every route is prerendered to a real file, so
deep links are real files and unknown paths genuinely miss.

---

## Cloudflare Pages build fails with "no package.json"

**Cause.** **Root directory** was not set. It is not part of any framework preset, and presets
assume the repository root.

**Fix.** Set it to the folder containing `package.json` — `site` for Graphite Hub. The build
output directory is relative to it, so it stays `dist`, not `site/dist`.

---

## A vanity redirect does nothing

**Cause.** A Bulk Redirect **list** was created but no Bulk Redirect **rule**. The list is data; the
rule activates it.

Also check you are in Account Home rather than the zone — Bulk Redirects are account-level.

---

## The site shows an empty page to a crawler

**Symptom.** Fetching the page without JavaScript shows an empty root element and nothing else.

**Cause.** A client-rendered SPA. Everything is in the JavaScript bundle.

**Fix.** Graphite Hub prerenders every route at build time with `react-dom/server`. If this
reappears, `scripts/prerender.mjs` did not run — check the build script still chains all three
steps: client build, SSR build, prerender.

---

## An image name is rejected by GHCR

**Cause.** GHCR requires lowercase. A repository name with capitals produces an invalid image name.

**Fix.** Already handled — the build workflow lowercases the repository name. Name repositories in
lowercase anyway.

---

## Docker build cannot find a file in a sibling folder

**Cause.** The build context is the service folder, not the repository root. `server/Dockerfile`
cannot copy from `../shared`.

**Fix.** Either vendor what is shared into the service, or publish it as a package. If two services
genuinely need the same code, that is a signal it belongs in a package rather than a folder.

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
