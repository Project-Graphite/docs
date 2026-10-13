# Project Graphite recommendations

## In place

- The platform deploy path: Helm-rendered Compose for containers and direct upload for static
  sites, with values reviewed by pull request and a pinned image tag per deploy. Projects move onto
  it one by one with [operations/migrating-from-coolify.md](operations/migrating-from-coolify.md).
- Secret values moved to the owners-only, SOPS-encrypted `secrets` repository.
- Traffic reaches Traefik directly with Let's Encrypt; CI deploys through a signed agent, so SSH stays limited to your address.
- Idle demonstrations can sleep and wake on request; Message Lab does.
- A memory budget in `platform/deploy/host.yaml` is checked on every pull request.
- Strata backs up its database and files nightly.

## Next

1. Finish [operations/migrating-from-coolify.md](operations/migrating-from-coolify.md), then
   measure `docker stats` and set each service's `reserve` to its real use.
2. Back up Graphite Tracker, Message Lab and PixelPocket like Strata, and copy backups off the VPS
   once off-site storage and its cost are approved. Test a restore of each.
3. Add availability and certificate checks from outside the VPS, with a user agent the sleep filter
   ignores.
4. Add a Hub deployment-status view from `tags/` and the values: image tag, last rollout, health.
5. Give the Graphite Tracker its scheduled wake (`sleep.wake`) if it should sleep: its release
   alerts and digests only run while it is awake.
6. Plan the move in [plans/k3s-transition.md](plans/k3s-transition.md) when the VPS grows.

## Deliberately deferred

- High availability, a second server, a load balancer and managed databases exceed the current
  portfolio goal and budget.
- Preview environments for stateful projects should wait until cleanup and cost controls exist.
- Untrusted third-party workloads do not belong on this shared Docker host.
