# Monitoring

Everything runs on the VPS and on GitHub; no outside service is involved.

| What | Where | Catches |
| :--- | :--- | :--- |
| Usage | the console | CPU, memory and storage for the host and every container, with 30 days of history |
| Alerts | the console, every minute | host CPU over 90%, memory over 90%, swap over 80%, disk over 85%; a container stopped, unhealthy, over 90% of its CPU or memory limit, or restarted three times in ten minutes (noting an out-of-memory kill); a project with no verified backup in 26 hours or whose last backup failed; the Hub answering over 1% of requests with errors over five minutes, or failing `/health/ready` twice in a row |
| Uptime | `platform` → **Uptime**, every 10 minutes from GitHub | an enabled site or the deploy agent that is unreachable, answers 5xx, or has a certificate failing or expiring within 14 days; works when the whole VPS is down |
| Rollouts and uptime on the Hub | `project-graphite.com/status` | every rollout the agent reported, 30 days of the Hub's own 30-second probe of each live host, p95 latency |
| Hub errors | the Hub's `/admin`, and an email to the owner | every unhandled error, fingerprinted; the first of a new fingerprint emails `ALERTS_EMAIL`, at most once a day per fingerprint |
| Logs | the console's Logs page | every container's output, collected and kept 7 days, so it survives a redeploy; Docker itself rotates each container's log at 10 MB, five files |

Thresholds and durations are rules on the console's Alerts page; an owner can change or disable
each one.

## Getting alerts

Both the console and **Uptime** open an issue labelled `alert` in `platform` and close it on
recovery: `Alert: …` from the console, `Down: <host>` from **Uptime**. Watch `platform` (Watch →
Custom → Issues) to get them by email or in the GitHub app. The console also shows active alerts
on its Overview and Alerts tab, and sends each alert and resolution by email when `SMTP_URL`,
`ALERTS_FROM` and `ALERTS_EMAIL` are set; "Send a test" on the Alerts page checks the transport.

The console needs a token to open issues. Without one it still shows alerts, but opens no issues.

1. GitHub → Settings → Developer settings → Fine-grained tokens → Generate. Resource owner
   `Project-Graphite`, repository `platform` only, permission Issues: read and write.
2. Add `ALERTS_TOKEN=<token>` to `/etc/graphite/console.env` on the VPS and redeploy the console
   from its Project page or by re-running **Deploy** for `console`.

A condition must hold for its rule's duration to raise an alert, and clear for a minute to resolve
it, so a single slow health check does not page anyone. **Uptime** retries a failed host after 30
seconds before it counts.

## What is not covered

- The console's own outage shows only as `Down: console.project-graphite.com` when Cloudflare cannot
  reach it; Access answers for it otherwise. Its absence from the Overview is the other sign.
- The Hub's error alerts come from the Hub itself, so a Hub that cannot start sends none; the
  console's readiness rule and **Uptime** cover that case.
