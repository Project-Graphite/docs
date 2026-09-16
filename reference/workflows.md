# Shared workflows

The public `actions` repository contains reusable GitHub Actions workflows. Project workflows pin a
reviewed commit or version tag rather than `main`.

```text
pull request -> detect services -> checks -> image build without push
main push    -> detect services -> checks -> publish immutable GHCR images -> record platform tags
platform tag update -> validate registry -> sync public Hub registry
```

`reusable-deploy.yml` records tags only when the platform entry declares `kind: compose`. It does
not access Coolify or the VPS. This separation means repository CI can publish an artifact without
holding server credentials or triggering an unreviewed deployment.

Coolify pulls the prebuilt images declared by each project's production Compose file. Selecting
Deploy remains a reviewed operations action. A future deploy webhook may automate that step after
rollback, secret handling, backups, and resource bounds are proven.

Organisation defaults remain read-only. Only the image-build job receives `packages: write`, and
the GitHub App can update only repositories explicitly installed for its registry-sync purpose.
