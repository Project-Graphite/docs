# Overview

Graphite Hub, at `project-graphite.com`, is the front of the organisation: member pages, project
pages, releases, a blog and what is running right now. It is a server on the OVH VPS, like every
containerized project; one static site remains on Cloudflare Pages. `platform` deploys both.

```text
project push -> CI checks -> GHCR sha-<short> image -> IMAGE_TAG in platform deploy/<slug>/values
                                                              |
platform Deploy <---------------------------------------------+
  |-> compose: render deploy/<slug>/compose.yaml over shared/ fragments
  |            -> signed POST to the VPS deploy agent -> docker compose up --wait -> route -> DNS
  |            -> the agent reports the rollout to the Hub
  `-> pages:   build the recorded commit -> Cloudflare Pages

visitor -> Cloudflare proxy -> VPS :443 -> Traefik -> graphite-hub        (the apex and the console)
visitor -> DNS-only record  -> VPS :443 -> Traefik -> <slug>:<port>       (every other project)

console Secrets page -> /etc/graphite/<slug>.env on the VPS -> read by docker compose at rollout
```

| Who | Changes | Where |
| :--- | :--- | :--- |
| a contributor | how a project runs | `platform/deploy/<slug>/`, by pull request |
| a contributor | what the Hub shows about a project | the project's `.graphite.yml`, or the project's editor on the Hub, which commits to that file |
| a member | their own page and posts | the Hub, signed in with GitHub |
| CI | which build runs | `IMAGE_TAG` in `platform/deploy/<slug>/values` |
| an owner | secret values | the console's Secrets page |
| an owner | every number, order and label on the Hub | `/admin` on the Hub |

Each containerized project gets:

- prebuilt GHCR images, health checks, restart behaviour and explicit resource limits from the
  shared fragments
- named volumes for persistent state, backed up nightly and verified when labelled for it
- secret values that never leave the console and the VPS
- one public service, reachable only through Traefik
- a deploy only after its CI passes, pinned to the image tag CI wrote
- a rollout record on the Hub's status page within a second of finishing

Projects share one Docker host, so this is operational separation for trusted projects, not a
security boundary. The 4 GB VPS is a portfolio host, not a high-availability platform.
