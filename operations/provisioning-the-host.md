# Provisioning the host

> **Designed, not provisioned.** Nothing below exists yet. The design is settled and the shape is
> accurate, but this is a plan, not a runbook you can follow today.

Until a host exists, `kind: compose` projects build and push images and record their tags
correctly, and nothing serves their hostname. Only `kind: pages` projects are actually live.

## When to build this

When the first project needs a backend. Not before. Until then the whole organisation costs the
price of the domain, and a box with nothing on it is a box to maintain for no reason.

## What it is

```
cloudflared ──▶ traefik ──▶ project containers
                              │
                          postgres
```

Four pieces: a tunnel, a router, a reconcile agent and a shared database.

`cloudflared` dials **out** to Cloudflare, so the box has no inbound ports, no exposed IP address
and no certificates to renew. Cloudflare terminates TLS; Traefik speaks plain HTTP on an internal
Docker network. That removes cert-manager, Let's Encrypt and firewall rules from the design
entirely.

Traefik routes by `Host` header, which is why one configuration covers every project and adding a
project needs no infrastructure change.

## The box

| Option | Cost | Notes |
| :--- | :--- | :--- |
| Hetzner CX22 | about 4 EUR/month | 2 vCPU, 4 GB, 40 GB, x86. Predictable. |
| Oracle Always Free | free | 4 ARM cores, 24 GB. Capacity is often unavailable; ARM means multi-architecture image builds. |

Start with Hetzner unless the 4 EUR matters. The ARM tax on Python images under QEMU is real.

The box is disposable by design: everything is in git and in container images, so rebuilding it is
a script, not an archaeology exercise. That is what makes a cheap or flaky host acceptable.

## Layout

```
/srv/graphite/
├── platform/           read-only clone of the platform repository
├── deploy/
│   ├── _edge/          traefik and cloudflared
│   ├── _data/          postgres and its backup job
│   └── <slug>/         generated per project
├── secrets/
│   └── <slug>.env      not in git, root-owned, mode 600
└── reconcile.sh
```

## The reconcile loop

A systemd timer runs every 60 seconds:

```sh
cd /srv/graphite/platform
git fetch --quiet origin main
[ "$(git rev-parse HEAD)" = "$(git rev-parse origin/main)" ] && exit 0
git reset --hard origin/main
for dir in deploy/*/; do
  slug=$(basename "$dir")
  docker compose -p "$slug" -f "$dir/compose.yaml" pull --quiet
  docker compose -p "$slug" -f "$dir/compose.yaml" up -d --remove-orphans
done
docker image prune -af --filter "until=168h"
```

Properties worth keeping:

- **Pull-based.** No SSH keys in CI, no inbound access, contributors never touch the box.
- **Git is the source of truth.** Rollback is `git revert` in `platform`; the box catches up.
- Same model as a GitOps controller, without running one.

## Build order

1. Provision the box, install Docker, create a non-root user, disable password SSH.
2. Create the Cloudflare tunnel, put its credentials on the box, bring up `_edge`.
3. Bring up `_data`: Postgres with a volume, plus a nightly `pg_dump` to Cloudflare R2.
4. Write the compose generator in `platform` and render one project.
5. Install `reconcile.sh` and the systemd timer.
6. Add the Terraform for per-project DNS records.
7. Deploy one real project end to end before trusting any of it.

## Security posture

Generated compose gives every service:

```yaml
restart: unless-stopped
read_only: true
security_opt: ["no-new-privileges:true"]
cap_drop: ["ALL"]
user: "10001:10001"
mem_limit: 512m
cpus: 0.5
tmpfs: ["/tmp"]
```

**Contributors never write production compose.** It is generated from the registry, so nobody can
mount the Docker socket or the host root, whether by mistake or otherwise. That is the main reason
the generator exists rather than each project shipping its own production compose file.

## Backups

Postgres dumps nightly to Cloudflare R2 — 10 GB free, no egress charges. The box itself is not
backed up and does not need to be: rebuilding is a script.

**Test the restore before you need it.** An untested backup is a guess.
