# Shared workflows

The public `actions` repository contains reusable GitHub Actions workflows. Projects call them at
`@main`; a repository that must hold back pins a commit SHA at the call site.

`reusable-pipeline.yml` chains detect, checks, images, deploy, changelog, gitleaks and PR
conventions behind boolean switches (`all`, `checks`, `deploy`, `changelog`, `secrets-scan`,
`conventions`; all `true` except `all`). The template's `pipeline.yml` calls it for pushes to `main`
and pull requests. Using it is optional: a repository can call the individual workflows instead.

```text
template pull request -> checks and image build without push
project main push     -> checks -> sha-<short> and latest images -> Deploy
Deploy                -> Coolify webhook when COOLIFY_WEBHOOK is set -> tag record in platform/tags
Coolify               -> pull latest images -> migrate -> health check -> replace containers
project main push     -> Changelog -> next version tag -> releases/<slug>/<version>.yml in graphite-hub
```

`reusable-deploy.yml` acts only when the platform entry declares `kind: compose`. It sends the
`POST` to the repository's Coolify webhook when the `COOLIFY_WEBHOOK` and `COOLIFY_TOKEN` secrets
exist, then records the tag in `platform/tags/<slug>.json`. Each compose repository holds its own
resource-specific webhook and deploy-only token in GitHub Actions secrets.

A service's `.build-contexts` file adds named build contexts, one `name=path` per line with the
path relative to the repository root. The build passes them to Docker, and change detection
rebuilds the service when any of those paths change. PixelPocket's frontend uses
`emulator=./gb-emulator`.

Coolify pulls prebuilt images declared by the production Compose file. Direct Coolify Auto Deploy
stays disabled so a repository push cannot race the image build. The workflow calls the webhook
only after its test and image jobs succeed. Manual Deploy remains available for first deployment,
configuration changes, and rollback.

Organisation defaults remain read-only. Only the image-build job receives `packages: write`. The
`graphite-deploy` GitHub App is installed on every organisation repository with contents and
pull-request write, because the registry sync reads every `.graphite.yml`.
