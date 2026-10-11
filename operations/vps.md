# The VPS

| Item | Value |
| :--- | :--- |
| Provider and plan | OVHcloud VPS-1, USD 5.35 a month at checkout |
| CPU, memory, disk | 2 vCores, about 3.7 GiB visible, about 38 GB |
| Operating system | Ubuntu 24.04 LTS, 2 GiB emergency swap |
| Hostname, IPv4 | `vps-2283122d`, `135.125.254.59` |
| Admin SSH | `ssh -i ~/.ssh/graphite-ovh ubuntu@135.125.254.59` |
| Deploy SSH | `ssh.project-graphite.com` through the tunnel and Cloudflare Access, user `deploy` |
| Projects | `/srv/graphite/<slug>/`: `compose.yaml`, `secrets.env`, `sleep` |
| Edge | `/srv/graphite/edge/`: Traefik, Sablier, cloudflared |

Renewing, resizing or adding an OVH option is a billing decision; check the current invoice first.

## What runs

```text
visitor -> Cloudflare -> tunnel -> cloudflared -> Traefik 127.0.0.1:8080 -> <slug>:<port> on the edge network
                                                           |
                                                           +-> Sablier wakes sleeping projects
```

- **Traefik** reads one route file per project from `/srv/graphite/edge/routes/`.
- **Sablier** stops a sleeping project's containers after `sleep.after` and starts them on the next
  request, showing a waiting page. Requests from bots and monitors get `200` without waking it.
- **cloudflared** runs the tunnel. No web port is open on the VPS; only SSH remains public, for
  your admin key.
- **Ofelia** (`/srv/graphite/wake/`) runs only when a project sets `sleep.wake`.

## Bootstrap

Done once, before the first platform deploy. Prerequisites are in
[migrating-from-coolify.md](migrating-from-coolify.md#1-prepare).

```sh
curl -fsSLO https://raw.githubusercontent.com/project-graphite/platform/main/host/bootstrap.sh
sudo TUNNEL_TOKEN='<tunnel token>' sh bootstrap.sh '<deploy public key>'
```

It creates the `deploy` user in the `docker` group with only that key, the `/srv/graphite` tree and
the `edge` Docker network, installs `graphite-up`, and starts the edge.

## Everyday commands

```sh
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
sudo docker stats --no-stream
sudo docker compose -p <slug> logs --tail 100 <service>
sudo -u deploy /srv/graphite/bin/graphite-up <slug>
df -h /
```

`graphite-up <slug>` re-applies a project exactly as last deployed. Prefer re-running the platform
**Deploy** workflow, which also refreshes the route and DNS.

## Firewall

The OVH edge firewall allows TCP 22 from your address, and denies everything else inbound once the
migration is finished. Ports 80 and 443 stay open only while Coolify still serves a project.

## Capacity

`platform/deploy/host.yaml` holds the budget, and the platform **PR** check fails when the projects'
`reserve` total exceeds it. Review capacity when steady memory exceeds 75%, swap grows, a container
is OOM-killed, or free disk drops toward 10 GB. Swap is an emergency buffer, not working memory.

## Updates

```sh
sudo apt update && sudo apt full-upgrade
sudo reboot
```

After a reboot, every project restarts on its own; Sablier stops the sleeping ones again.
