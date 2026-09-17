# Shared workflows

The public `actions` repository contains reusable GitHub Actions workflows. Project workflows pin a
reviewed commit or version tag rather than `main`.

```text
pull request -> checks -> image build without push
main push    -> tests -> publish SHA and main images -> record tag -> authenticated Coolify webhook
Coolify      -> pull main images -> migrate -> health check -> replace containers
platform tag update -> validate registry -> sync public Hub registry
```

`reusable-deploy.yml` records tags only when the platform entry declares `kind: compose`. It does
not access Coolify or the VPS. The project workflow separately holds a resource-specific Coolify
webhook and deploy-only token in GitHub Actions secrets.

Coolify pulls prebuilt images declared by the production Compose file. Direct Coolify Auto Deploy
stays disabled so a repository push cannot race the image build. The workflow calls the webhook
only after its test and image jobs succeed. Manual Deploy remains available for first deployment,
configuration changes, and rollback.

Organisation defaults remain read-only. Only the image-build job receives `packages: write`, and
the GitHub App can update only repositories explicitly installed for its registry-sync purpose.
