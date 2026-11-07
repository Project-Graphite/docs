# The VPS

| Item | Value |
| :--- | :--- |
| Provider and plan | OVHcloud VPS-1, USD 5.35 a month at checkout |
| CPU, memory, disk | 2 vCores, about 3.7 GiB visible, about 38 GB |
| Operating system | Ubuntu 24.04 LTS, 2 GiB emergency swap |
| Hostname, IPv4 | `vps-2283122d`, `135.125.254.59` |
| Admin SSH | `ssh -i ~/.ssh/graphite-ovh ubuntu@135.125.254.59`, port 22 open only to your IP |
| Projects | `/srv/graphite/<slug>/`: `compose.yaml`, `values`, `files/`, and `.applied` from the last rollout |
| Secrets | `/etc/graphite/<slug>.env`, root, mode 600, written by the console; `console.env` placed by hand; `agent.env` for the agent |
| Edge | `/srv/graphite/edge/`: Traefik with Let's Encrypt, Sablier, one route file per project under `routes/` |
| Backups | `/var/lib/graphite/backups/<project>/<stamp>.tar` with a `.json` summary each; `audit.log`; `.schedule.json` |
| Deploy agent | `graphite-agent` systemd service, reached at `https://deploy.project-graphite.com` |
| Console | `https://console.project-graphite.com`, see [console.md](console.md) |

Renewing, resizing or adding an OVH option is a billing decision; check the current invoice first.

## What runs

```text
visitor -> Cloudflare proxy  -> VPS :443 -> Traefik -> graphite-hub, console
visitor -> DNS-only A record -> VPS :443 -> Traefik -> <slug>:<port> on the edge network
platform Deploy -> https://deploy.project-graphite.com -> agent -> graphite-up -> docker compose up --wait
                                                                       `-> graphite-report -> the Hub
console Secrets page -> /etc/graphite/<slug>.env;  console Redeploy -> graphite-up
backup service -> nightly archives under /var/lib/graphite/backups, verified; the console orders runs and restores
```

- **Traefik** serves ports 80 and 443, redirects HTTP to HTTPS, gets certificates from Let's
  Encrypt through a Cloudflare DNS challenge, and reads one route file per project from
  `/srv/graphite/edge/routes/`.
- **The deploy agent** (`/srv/graphite/bin/agent.py`) accepts requests signed with `DEPLOY_TOKEN`:
  it refuses a compose file that asks for host access, refuses a rollout whose required secret
  names are missing from `/etc/graphite/<slug>.env`, writes the project's files and runs
  `graphite-up`. It listens only on the Docker bridge, behind Traefik.
- **`graphite-up <slug>`** runs `docker compose up -d --wait` with `values` and the env file,
  touches `.applied`, prunes images older than a day, and on exit runs `graphite-report`, which
  posts the rollout and the host's health to the Hub's `/internal/deploys`, signed with
  `DEPLOY_REPORT_SECRET` from `/etc/graphite/agent.env`. The Hub shows it within a second.
- **The backup service** (`backup.py` from `platform/edge/`, the `backup` compose project) runs
  the schedule and the orders the console leaves under `/var/lib/graphite/backups/.requests/`.
- **Sablier** still runs in the edge, but no route uses it.

## Bootstrap

Once, on a fresh VPS with Docker Compose and `python3`, as `ubuntu`:

```sh
curl -fsSLO https://raw.githubusercontent.com/project-graphite/platform/main/host/bootstrap.sh
sudo DEPLOY_TOKEN='<deploy token>' CF_DNS_API_TOKEN='<certificate token>' sh bootstrap.sh
```

It creates the `deploy` system user in the `docker` group and the `/srv/graphite` tree, installs
`graphite-up`, `graphite-report`, the agent and the edge files, writes `/etc/graphite/agent.env`
with `DEPLOY_TOKEN` and `BIND`, creates the `edge` network, enables `graphite-agent.service`, then
runs `graphite-up edge`. The first certificate needs the certificate token to have Zone → Zone →
Read as well as Zone → DNS → Edit. Run it again to update the agent and the host scripts; nothing
else copies `host/` files to the VPS. Then place `/etc/graphite/console.env` by hand and deploy
the console, which writes every other project's env file from its store.

## Everyday commands

```sh
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
sudo docker stats --no-stream
sudo docker compose -p <slug> logs --tail 100 <service>
sudo -u deploy /srv/graphite/bin/graphite-up <slug>
sudo journalctl -u graphite-agent --since today
sudo docker compose -p backup logs --tail 50
ls -l /var/lib/graphite/backups/<slug>/
df -h /
```

`graphite-up <slug>` re-applies a project exactly as last deployed, with the current env file; the
console's Redeploy button does the same. Prefer re-running the platform **Deploy** workflow, which
also refreshes the route and DNS. The console shows most of this in the browser.

## Firewall

The OVH edge firewall allows TCP 22 from your address, TCP 80, TCP 443 and UDP 443, and denies
everything else inbound.

## Capacity

`platform/deploy/host.yaml` holds the budget, 3700M minus 700M for the system, and the platform
**PR** check fails when the projects' reservations exceed it. Review capacity when steady memory
exceeds 75%, swap grows, a container is OOM-killed, or free disk drops toward 10 GB. Swap is an
emergency buffer, not working memory. Backups take disk too: 14 days of every labelled database
and volume under `/var/lib/graphite/backups`.

## Updates

```sh
sudo apt update && sudo apt full-upgrade
sudo reboot
```

After a reboot every project restarts on its own and the agent starts with Docker.
