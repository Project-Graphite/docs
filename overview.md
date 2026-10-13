# Overview

Graphite Hub publishes member portfolios and project case studies at `project-graphite.com`.
Every project is deployed from `platform`: static sites to Cloudflare Pages, containerized projects
as Docker Compose projects on one OVH VPS, served directly with Let's Encrypt.

```text
project push -> CI checks -> GHCR sha-<short> image -> platform tags/<slug>.json
                                                              |
platform Deploy <---------------------------------------------+
  |-> compose: render deploy/<slug>/values.yaml with the core chart
  |            -> signed POST to the VPS deploy agent -> docker compose up --wait -> route -> DNS
  `-> pages:   build the recorded commit -> Cloudflare Pages

visitor -> VPS :443 -> Traefik -> project (Sablier wakes it if it sleeps)
```

| Who | Changes | Where |
| :--- | :--- | :--- |
| a contributor | how a project runs | `platform/deploy/<slug>/values.yaml`, by pull request |
| a contributor | what the Hub shows | the project's `.graphite.yml` |
| CI | which build runs | `platform/tags/<slug>.json` |
| an owner | secret values | the `secrets` repository |

Each containerized project gets:

- prebuilt GHCR images, health checks, restart behaviour and explicit resource limits
- named volumes for persistent state
- secret values that never leave the `secrets` repository and the VPS
- one public service, reachable only through Traefik
- a deploy only after its CI passes, pinned to an image tag unless its values pin another
- optional sleep: stopped when idle, started on the next request

Projects share one Docker host, so this is operational separation for trusted projects, not a
security boundary. The 4 GB VPS is a portfolio host, not a high-availability platform.
