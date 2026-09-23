# Shared workflows

The public `actions` repository contains reusable GitHub Actions workflows. Projects call them at
`@main`; a repository that must hold back pins a commit SHA at the call site.

```text
template pull request -> checks and image build without push
template main push    -> sha-<short> and latest images -> tag record for compose entries
Message Lab main push -> tests -> sha-<commit> and main images -> Coolify webhook
                                                               -> tag record when GRAPHITE_DEPLOY_ENABLED is true
Message Lab Coolify   -> pull main images -> migrate -> health check -> replace containers
```

`reusable-deploy.yml` records tags only when the platform entry declares `kind: compose`. It does
not access Coolify or the VPS. Message Lab's workflow separately holds a resource-specific Coolify
webhook and deploy-only token in GitHub Actions secrets.

Coolify pulls prebuilt images declared by the production Compose file. Direct Coolify Auto Deploy
stays disabled so a repository push cannot race the image build. The workflow calls the webhook
only after its test and image jobs succeed. Manual Deploy remains available for first deployment,
configuration changes, and rollback.

Organisation defaults remain read-only. Only the image-build job receives `packages: write`. The
`graphite-deploy` GitHub App is installed on every organisation repository with contents and
pull-request write, because the registry sync reads every `.graphite.yml`.
