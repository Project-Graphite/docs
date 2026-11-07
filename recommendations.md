# Project Graphite recommendations

## In place

- The platform deploy path: plain Compose over shared fragments, reviewed by pull request, applied
  through a signed agent, with the image tag CI wrote. Every project deploys from `platform`, and
  every rollout is on the Hub's status page.
- Secret values in the console's store, written to the VPS, never in git or CI.
- The apex and the console behind Cloudflare; every other host reaches Traefik directly with Let's
  Encrypt; SSH stays limited to your address.
- One backup service for every labelled database and volume, nightly, verified, kept 14 days,
  downloadable and importable from the console.
- A memory budget in `platform/deploy/host.yaml` checked on every pull request.
- Alerts from the console and **Uptime** as GitHub issues, with the Hub's error rate and readiness
  among them.
- The Hub as a server: member-edited pages, a co-edited blog, subscriptions, and live updates on
  every page.

## Next

1. Measure `docker stats` and set each service's `*_RESERVE` to its real use.
2. Copy backups off the VPS once off-site storage and its cost are approved; the Backups page's
   download is the manual path until then.
3. Label Graphite Tracker's and PixelPocket's data for backup if any of it is not yet, and run an
   import of each from the Backups page once.
4. Delete the retired `graphite-hub` Cloudflare Pages project and the `secrets` repository's
   archive once nothing refers to them.
5. Move the Hub's email to a dedicated provider before digests approach Gmail's daily limit.
6. Plan the move in [plans/k3s-transition.md](plans/k3s-transition.md) when the VPS grows; the plan
   predates the plain compose path and needs rewriting first.

## Deliberately deferred

- High availability, a second server, a load balancer, a staging host and managed databases exceed
  the current portfolio goal and budget.
- Preview environments for stateful projects should wait until cleanup and cost controls exist.
- Untrusted third-party workloads do not belong on this shared Docker host.
- Comments, mentions and trending on the Hub; each would need a notification centre or moderation
  it does not have.
