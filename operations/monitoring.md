# Monitoring

Everything runs on the VPS and on GitHub; no outside service is involved.

| What | Where | Catches |
| :--- | :--- | :--- |
| Usage | the console | CPU, memory and storage for the host and every container, with 30 days of history |
| Alerts | the console, every minute | a container that stopped, is unhealthy or keeps restarting (with out-of-memory kills); host memory over 90%, swap over 80%, disk over 85%; a `backup` with no dump in 26 hours |
| Uptime | `platform` → **Uptime**, every 10 minutes from GitHub | a site or the deploy agent that is unreachable, answers 5xx, or has a certificate failing or expiring within 14 days; works when the whole VPS is down |
| Logs | every container | rotated at 10 MB, five files kept, so they cannot fill the disk |

## Getting alerts

Both the console and **Uptime** open an issue labelled `alert` in `platform` and close it on
recovery: `Alert: …` from the console, `Down: <host>` from **Uptime**. Watch `platform` (Watch →
Custom → Issues) to get them by email or in the GitHub app. The console also shows active alerts
on its Overview and Alerts tab.

The console needs a token to open issues. Without one it still shows alerts, but opens no issues.

1. GitHub → Settings → Developer settings → Fine-grained tokens → Generate. Resource owner
   `Project-Graphite`, repository `platform` only, permission Issues: read and write.
2. Add `ALERTS_TOKEN=<token>` to `console.env` in `secrets` with `sops edit console.env` and push.
   **Apply** restarts the console with it.

A condition must hold for two checks in a row to raise an alert, and clear for two to resolve it,
so a single slow health check does not page anyone. **Uptime** retries a failed host after 30
seconds before it counts.

## What is not covered

- Logs are kept per container and go when it is recreated by a deploy. Read them in the console
  before redeploying something you are investigating.
- The console's own outage shows only as `Down: console.project-graphite.com` when Cloudflare cannot
  reach it; Access answers for it otherwise. Its absence from the Overview is the other sign.
