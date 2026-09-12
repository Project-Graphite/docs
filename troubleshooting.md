# Troubleshooting

Every entry here is a failure that actually happened while building this, with the cause and the
fix. Most of them present with an error pointing somewhere other than the problem.

---

## A workflow run fails with no jobs and no log

**Symptom.** The run is marked failed. Job count is zero. There is no annotation, no log to
download, and the workflow name in the Actions list shows as the file path instead of its `name:`.

That last detail is the tell: **GitHub could not parse or resolve the workflow**, so it never got
as far as reading the name. It is a startup failure, not a runtime one.

Two causes, and `actionlint` catches neither.

### Cause 1: the caller did not grant the permissions the called workflow needs

A reusable workflow cannot request more than its caller grants. The organisation leaves
`GITHUB_TOKEN` read-only, so a workflow declaring `packages: write` fails at startup unless the
calling job grants it.

```yaml
  build:
    permissions:
      contents: read
      packages: write
    uses: project-graphite/actions/.github/workflows/reusable-docker-build-push.yml@v1
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
gh secret set PLATFORM_APP_ID --repo project-graphite/platform --body "4923055"
gh secret set PLATFORM_APP_PRIVATE_KEY --repo project-graphite/platform < key.pem
```

Public repositories pick up organisation secrets automatically, which is why project repositories
are public and members cannot create private ones.

---

## Deploy fails with GH013: Repository rule violations

**Symptom.** The deploy job mints a token, checks out the target repository, then fails on push
with `GH013: Repository rule violations found for refs/heads/main`.

**Cause.** A ruleset on the target repository requires a pull request, and the GitHub App is not a
bypass actor. `RepositoryRole: 5` covers repository admins — that is you, not the app.

**Fix.** Add the app by App ID as an `Integration` bypass actor. See
[operations/rotating-the-deploy-app.md](operations/rotating-the-deploy-app.md).

Any new ruleset on a repository the bot writes to needs this, or deploys start failing in a way
that looks like a permissions problem.

---

## Deploy fails with "No registry entry"

**Symptom.** Images build and push fine; the deploy job fails saying `projects/<slug>.yml` does not
exist.

**Cause.** Working as intended. A repository cannot deploy until it is registered.

**Fix.** Add the registry entry. Until then builds and checks still run and only the record step is
blocked, deliberately, so a new repository does not silently do nothing.

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

**Symptom.** Validation fails with `name: 2048 is not of type 'string'`.

**Cause.** YAML parses a bare `2048` as an integer.

**Fix.** Quote it: `name: "2048"`. Applies to anything that looks numeric — versions, years, values
with leading zeros.

---

## A shell script fails on Linux with "bad interpreter"

**Cause.** The file was committed with CRLF line endings from a Windows checkout.

**Fix.** Every repository has `.gitattributes` with `* text=auto eol=lf`, inherited from the
template. If a repository was created some other way, add it. Verify what is actually committed:

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

**Fix.** Ship a `404.html`. The portfolio has one in `site/public/`, so it lands in `dist/`.

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

The portfolio does not rely on SPA fallback at all. Every route is prerendered to a real file, so
deep links are real files and unknown paths genuinely miss.

---

## Cloudflare Pages build fails with "no package.json"

**Cause.** **Root directory** was not set. It is not part of any framework preset, and presets
assume the repository root.

**Fix.** Set it to the folder containing `package.json` — `site` for the portfolio. The build
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

**Fix.** The portfolio prerenders every route at build time with `react-dom/server`. If this
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
