# Managing secrets

Secret values live only in the private `secrets` repository, which only organisation owners can see
or change. Each project's values in `platform` name the keys it needs; an owner adds the values.

| You want to | Do |
| :--- | :--- |
| add or change a value | `sops edit <slug>.env` in `secrets`, commit, push |
| apply a file again | `secrets` → Actions → **Apply** → Run workflow with the slug |
| see which keys a project needs | `secrets:` and the `${NAME}` references in `platform/deploy/<slug>/values.yaml` |
| see what the VPS has | `sudo cut -d= -f1 /srv/graphite/<slug>/secrets.env` |

A push that changes `<slug>.env` decrypts it in **Apply**, writes
`/srv/graphite/<slug>/secrets.env` over SSH, and restarts that project. A deploy refuses to run
while a name listed in `secrets:` is missing and reports `waiting on secrets: …`.

`edge.env` holds `TUNNEL_TOKEN` for cloudflared.

## Keys

| Key | Where |
| :--- | :--- |
| age private key | your password manager, `%AppData%\sops\age\keys.txt`, `secrets` → `SOPS_AGE_KEY` |
| age public key | `secrets/.sops.yaml` |
| deploy SSH key | your password manager, `DEPLOY_SSH_KEY` in `platform` (`production`) and `secrets` |
| Access service token | `CF_ACCESS_CLIENT_ID`/`SECRET` in `platform` (`production`) and `secrets` |
| Cloudflare API token | `CLOUDFLARE_API_TOKEN` in `platform` (`production`) |

`platform`'s `production` environment only runs from `main`, so a pull request branch cannot read
them.

## Rotating

- **A project value:** edit it, push, and **Apply** restarts the project.
- **The age key:** `age-keygen`, replace the public key in `.sops.yaml`, run
  `sops updatekeys --yes *.env`, replace `SOPS_AGE_KEY`, push, then destroy the old key.
- **The deploy SSH key:** add the new public key to `/home/deploy/.ssh/authorized_keys`, replace
  `DEPLOY_SSH_KEY` in both repositories, run **Deploy** with `edge`, then remove the old public key.
- **The Access service token or API token:** create the new one in Cloudflare, replace the
  secrets, run **Deploy** with `edge`, then revoke the old one.

If the age key leaks, every value in the repository's history is readable: rotate the key and every
project value.
