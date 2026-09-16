# Shared workflows

The public `actions` repository contains reusable GitHub Actions workflows. Project workflows pin a
reviewed commit or version tag rather than `main`.

```text
pull request -> detect services -> checks -> image build without push
main push    -> detect services -> checks -> publish immutable GHCR images -> record platform tags
platform tag update -> validate registry -> render Kubernetes YAML -> sync public Hub registry
```

`reusable-deploy.yml` records tags only when the platform entry declares `kind: kubernetes`. It does
not access the cluster. This separation means repository CI can publish an artifact without holding
Google Cloud or Kubernetes credentials.

The current platform workflow renders manifests as a validation gate. Applying them remains a
reviewed operations action. A future pull-based reconciler may automate that step once the lab has
proven rollback, secret handling, and resource bounds.

Organisation defaults remain read-only. Only the image-build job receives `packages: write`, and
the GitHub App can update only repositories explicitly installed for its registry-sync purpose.
