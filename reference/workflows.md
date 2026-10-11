# Shared workflows

The public `actions` repository holds reusable workflows and the `vps-ssh` action. Projects call
them at `@main`; a repository that must hold back pins a commit SHA at the call site.

`reusable-pipeline.yml` chains detect, checks, images, deploy, changelog, gitleaks and PR
conventions behind boolean switches (`all`, `checks`, `deploy`, `changelog`, `secrets-scan`,
`conventions`; all `true` except `all`). The template's `pipeline.yml` calls it. Using it is
optional: a repository can call the individual workflows instead.

```text
pull request     -> checks -> image build without push
main push        -> checks -> sha-<short> and latest images -> reusable-deploy -> platform tags/<slug>.json
platform push    -> Deploy -> render values -> SSH -> docker compose up --wait -> route -> DNS
                             -> or build the site -> Cloudflare Pages
main push        -> Changelog -> next version tag -> releases/<slug>/<version>.yml in graphite-hub
secrets push     -> Apply -> secrets.env on the VPS -> restart the project
```

`reusable-deploy.yml` records the tag only when the registry entry is `compose` or `pages`. A
static site records `["site"]` with the pushed commit. While a project still runs on Coolify, its
`COOLIFY_WEBHOOK` and `COOLIFY_TOKEN` secrets make the same workflow call Coolify.

A service's `.build-contexts` file adds named build contexts, one `name=path` per line relative to
the repository root. The build passes them to Docker, and change detection rebuilds the service
when any of those paths change. PixelPocket's frontend uses `emulator=./gb-emulator`.

Organisation defaults remain read-only. Only the image build receives `packages: write`. The
`graphite-deploy` GitHub App is installed on every repository with contents and pull-request write,
because the registry sync reads every `.graphite.yml` and CI writes tags into `platform`. Deploy
credentials live in `platform`'s `production` environment, which only `main` can use.
