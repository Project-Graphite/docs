# GKE migration

## Outcome

Move Project Graphite's deployable-project contract from a shared Docker Compose host to one shared,
zonal GKE Standard cluster while retaining Docker Compose for local development.

## Changes

- Replace the platform registry's `compose` deployment kind with `kubernetes`.
- Add per-project namespace, resource, security and routing contracts.
- Keep immutable GHCR image recording compatible with Kubernetes projects.
- Add reusable Kubernetes scaffolding to the project template.
- Add a complete Message Lab Kubernetes deployment and remove its VM deployment files.
- Update Hub content and all hosting, operations and recommendation documentation.
- Prepare reviewed Google Cloud commands without running them.

## Verification

- Validate and compile the platform registry.
- Validate Kubernetes YAML structure and render the Message Lab manifests.
- Run existing Message Lab and Hub checks.
- Validate workflow YAML and review all repository diffs.
- Confirm no Google Cloud command was executed.
