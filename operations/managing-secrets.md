# Managing secrets

Secret values live only in the private `secrets` repository, which only organisation owners can see
or change. Each project's values in `platform` name the keys it needs; an owner adds the values.

| You want to | Do |
| :--- | :--- |
| add or change a value | `sops edit <slug>.env` in `secrets`, commit, push |
| apply a file again | `secrets` → Actions → **Apply** → Run workflow with the slug |
| see which keys a project needs | each service's `secrets` in `platform/deploy/<slug>/values.yaml`, plus `POSTGRES_PASSWORD` with `postgres` and `REDIS_PASSWORD` with `redis: {password: true}` |
| see what the VPS has | `sudo cut -d= -f1 /srv/graphite/<slug>/secrets.env` |

A push that changes `<slug>.env` decrypts it in **Apply**, writes
`/srv/graphite/<slug>/secrets.env` through the deploy agent, and restarts that project. A deploy refuses to run
while a name listed in `secrets:` is missing and reports `waiting on secrets: …`.

`edge.env` holds `CF_DNS_API_TOKEN`, the token Traefik uses for certificates. It needs Zone → Zone →
Read and Zone → DNS → Edit on `project-graphite.com`.

## Keys

| Key | Where |
| :--- | :--- |
| age private key | your password manager, `%AppData%\sops\age\keys.txt`, `secrets` → `SOPS_AGE_KEY` |
| age public key | `secrets/.sops.yaml` |
| deploy token | your password manager, `/etc/graphite-agent.env` on the VPS, `DEPLOY_TOKEN` in `platform` (`production`) and `secrets` |
| Cloudflare API token | `CLOUDFLARE_API_TOKEN` in `platform` (`production`); Zone → DNS → Edit on `project-graphite.com` and Account → Cloudflare Pages → Edit |
| certificate DNS token | `secrets/edge.env` |

`platform`'s `production` environment only runs from `main`, so a pull request branch cannot read
them. **Deploy** also reads the `platform` variables `DEPLOY_URL`
(`https://deploy.project-graphite.com`), `VPS_IPV4` (`135.125.254.59`), `CLOUDFLARE_ACCOUNT_ID`
and `CLOUDFLARE_ZONE_ID`.

## Creating the Cloudflare tokens

Cloudflare → My Profile → API Tokens → Create Token → Custom token, both limited to the
`project-graphite.com` zone:

| Token | Permissions | Goes in |
| :--- | :--- | :--- |
| platform | Zone → DNS → Edit; Account → Cloudflare Pages → Edit | `CLOUDFLARE_API_TOKEN` in `platform`'s `production` environment |
| certificate | Zone → Zone → Read; Zone → DNS → Edit | `secrets/edge.env` and the bootstrap command |

The account and zone IDs, on the domain's overview page, are the `CLOUDFLARE_ACCOUNT_ID` and
`CLOUDFLARE_ZONE_ID` variables in `platform`.

## Rotating

- **A project value:** edit it, push, and **Apply** restarts the project.
- **The age key:** `age-keygen`, replace the public key in `.sops.yaml`, run
  `sops updatekeys --yes *.env`, replace `SOPS_AGE_KEY`, push, then destroy the old key.
- **The deploy token:** generate a new one, put it in `/etc/graphite-agent.env` and run
  `sudo systemctl restart graphite-agent`, then replace `DEPLOY_TOKEN` in `platform` and `secrets`.
- **A Cloudflare token:** create the new one, replace `CLOUDFLARE_API_TOKEN` or `edge.env`, push or
  run **Deploy** with `edge`, then revoke the old one.

If the age key leaks, every value in the repository's history is readable: rotate the key and every
project value.
