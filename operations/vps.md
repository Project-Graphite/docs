# The VPS

| Item | Value |
| :--- | :--- |
| Provider and plan | OVHcloud VPS-1, USD 5.35 a month at checkout |
| CPU, memory, disk | 2 vCores, about 3.7 GiB visible, about 38 GB |
| Operating system | Ubuntu 24.04 LTS, 2 GiB emergency swap |
| Hostname, IPv4 | `vps-2283122d`, `135.125.254.59` |
| Admin SSH | `ssh -i ~/.ssh/graphite-ovh ubuntu@135.125.254.59`, port 22 open only to your IP |
| Projects | `/srv/graphite/<slug>/`: `compose.yaml`, `secrets.env`, `sleep` |
| Edge | `/srv/graphite/edge/`: Traefik with Let's Encrypt, Sablier; `secrets.env` holds the certificate DNS token |
| Deploy agent | `graphite-agent` systemd service, reached at `https://deploy.project-graphite.com` |
| Console | `https://console.project-graphite.com`, see [console.md](console.md) |

Renewing, resizing or adding an OVH option is a billing decision; check the current invoice first.

## What runs

```text
visitor -> DNS-only A record -> VPS :443 -> Traefik -> <slug>:<port> on the edge network
                                              |
                                              +-> Sablier wakes sleeping projects first
platform Deploy -> https://deploy.project-graphite.com -> agent -> docker compose up --wait
```

- **Traefik** serves ports 80 and 443, redirects HTTP to HTTPS, gets certificates from Let's
  Encrypt through a Cloudflare DNS challenge, and reads one route file per project from
  `/srv/graphite/edge/routes/`.
- **Sablier** stops a sleeping project after `sleep.after` and starts it on the next request,
  showing a waiting page. Requests from bots and monitors get `200` without waking it.
- **The deploy agent** (`/srv/graphite/bin/agent.py`) accepts requests signed with `DEPLOY_TOKEN`:
  it writes a project's files or secrets and runs `graphite-up`. It listens only on the Docker
  bridge, behind Traefik.
- **Ofelia** (`/srv/graphite/wake/`) runs only when a project sets `sleep.wake`.

## Bootstrap

Once, on a fresh VPS with Docker Compose and `python3`, as `ubuntu`:

```sh
curl -fsSLO https://raw.githubusercontent.com/project-graphite/platform/main/host/bootstrap.sh
sudo DEPLOY_TOKEN='<deploy token>' CF_DNS_API_TOKEN='<certificate token>' sh bootstrap.sh
```

It creates the `deploy` system user in the `docker` group and the `/srv/graphite` tree, installs
`graphite-up`, the agent and the edge files, writes `/etc/graphite-agent.env`, creates the `edge`
network, enables `graphite-agent.service`, then runs `graphite-up edge`. The first certificate
needs the certificate token to have Zone → Zone → Read as well as Zone → DNS → Edit. Run it again to
update the agent.

## Everyday commands

```sh
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
sudo docker stats --no-stream
sudo docker compose -p <slug> logs --tail 100 <service>
sudo -u deploy /srv/graphite/bin/graphite-up <slug>
sudo journalctl -u graphite-agent --since today
df -h /
```

`graphite-up <slug>` re-applies a project exactly as last deployed. Prefer re-running the platform
**Deploy** workflow, which also refreshes the route and DNS. The console shows most of this in the
browser.

## Firewall

The OVH edge firewall allows TCP 22 from your address, TCP 80, TCP 443 and UDP 443, and denies
everything else inbound.

## Capacity

`platform/deploy/host.yaml` holds the budget, and the platform **PR** check fails when the projects'
`reserve` total exceeds it. Review capacity when steady memory exceeds 75%, swap grows, a container
is OOM-killed, or free disk drops toward 10 GB. Swap is an emergency buffer, not working memory.

## Updates

```sh
sudo apt update && sudo apt full-upgrade
sudo reboot
```

After a reboot every project restarts on its own, the agent starts with Docker, and Sablier stops the
sleeping projects again.
