# CI reference

All CI lives in [`actions`](https://github.com/project-graphite/actions) as reusable workflows.
Project repositories do not define their own pipeline; they call these, so a change lands once and
reaches everything.

`actions` is **public** and must stay that way. A public repository cannot call a reusable workflow
in a private one — see [decisions.md](../decisions.md).

## Pinning

Call these at `@v1`, never `@main`:

```yaml
uses: project-graphite/actions/.github/workflows/reusable-checks.yml@v1
```

`v1` moves forward with backwards-compatible changes. A breaking change gets `v2` and repositories
move deliberately. Pinning to `@main` means one bad commit breaks CI everywhere at once.

## Permissions

The organisation leaves `GITHUB_TOKEN` read-only. A reusable workflow cannot request more than its
caller grants, so jobs calling a workflow that needs write access must grant it at the call site:

| Calling a workflow that | Grant |
| :--- | :--- |
| pushes images | `contents: read`, `packages: write` |
| labels pull requests | `contents: read`, `issues: write`, `pull-requests: write` |
| anything else | nothing |

Without the grant the run fails **at startup with no job log and no annotation**, which is an
unhelpful error for a misleading cause. If a workflow fails with zero jobs, check this first.

---

## The workflows

| Workflow | Purpose |
| :--- | :--- |
| `reusable-detect-services.yml` | Outputs the services changed in this event |
| `reusable-checks.yml` | Lint, test and build each changed service |
| `reusable-docker-build-push.yml` | Build each changed service, push to GHCR |
| `reusable-deploy.yml` | Record new image tags in `platform` |
| `reusable-pr-checks.yml` | PR title, size label, path labels |
| `reusable-gitleaks.yml` | Scan full history for committed secrets |

### reusable-detect-services

| | |
| :--- | :--- |
| Inputs | `all` (boolean, default `false`) — select every service rather than changed ones |
| Outputs | `services` — JSON array, e.g. `["frontend","server"]`<br>`any` — `"true"` or `"false"` |

On a pull request it compares against the base commit; on a push, against the previous commit. When
that commit is unreachable — a new branch, a force push — it selects everything.

### reusable-checks

| | |
| :--- | :--- |
| Inputs | `services` (required), `node-version` (default `22`), `python-version` (default `3.12`) |

Skips entirely when `services` is `[]`.

### reusable-docker-build-push

| | |
| :--- | :--- |
| Inputs | `services` (required), `push` (boolean, default `true`) |
| Outputs | `tag` — `sha-<short>`, applied to every image in the run |

Pushes `ghcr.io/project-graphite/<repo>/<service>` tagged `sha-<short>` and `latest`. Layer cache is
scoped per service, so one service rebuilding does not invalidate another.

The tag is computed in its own job rather than inside the build matrix, so the workflow output is
deterministic rather than whichever matrix leg finished last.

### reusable-deploy

| | |
| :--- | :--- |
| Inputs | `services` (required), `tag` (required) |
| Secrets | `PLATFORM_APP_ID`, `PLATFORM_APP_PRIVATE_KEY` (both optional) |

Merges the new tags into `platform/tags/<slug>.json` and pushes. Fails with a readable message if:

- the app secrets are missing — images still built and pushed, only the record is blocked
- there is no `projects/<slug>.yml` — the repository has not been registered

Skips silently when the registry entry declares a kind other than `compose`, since nothing else
runs container images.

Secrets are declared optional so a repository without them fails **at the deploy job** rather than
at workflow startup, which would take the build down with it.

### reusable-pr-checks

| | |
| :--- | :--- |
| Inputs | `xs-max` (10), `s-max` (50), `m-max` (200), `l-max` (600) |

The title check requires `type(scope): summary`. Size counts changed lines excluding lockfiles, so a
dependency bump is not labelled `size/xl`. Path labels apply only if the repository has a
`.github/labeler.yml`.

### reusable-gitleaks

| | |
| :--- | :--- |
| Inputs | `version` (default `8.30.1`) |

Runs the pinned release binary over full history. The marketplace action needs a paid licence for
organisation repositories; the binary does not.

---

## What a project repository contains

Two files, identical everywhere.

`.github/workflows/pr.yml` — detect, checks, build without pushing, gitleaks, conventions.

`.github/workflows/main.yml` — detect, build and push, record tags.

You should not need to edit either. If you find yourself wanting to, the change probably belongs in
`actions` so every project gets it.

---

## Linting workflows

`actions` lints itself with [actionlint](https://github.com/rhysd/actionlint) on every push, plus a
check that every action reference is pinned to a major version. Run it locally before pushing
workflow changes:

```sh
actionlint .github/workflows/*.yml
```

It catches YAML errors, invalid expressions and shellcheck issues in `run` blocks. It does **not**
catch the permissions problem above, which is a plan and policy issue rather than a syntax one.
