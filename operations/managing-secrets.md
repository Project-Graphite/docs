# Managing secrets

Secret values live in the console's store: encrypted in its SQLite, edited on its **Secrets** page,
and written to `/etc/graphite/<slug>.env` on the VPS, which `docker compose` reads at every
rollout. A project's `compose.yaml` in `platform` names what it needs as `${NAME:?}`; an owner, or
a member granted that project's secrets, supplies the values. Nothing passes through git or CI.

| You want to | Do |
| :--- | :--- |
| add or change a value | Secrets → the project → Create, or open the secret and Edit. The env file on the VPS is rewritten at once |
| add many at once | Create → Import: paste `NAME=value` lines, review the parsed names, create them |
| apply a change to a running project | the project shows **needs redeploy** on the Secrets page, the Project page and the Overview until you press Redeploy, which re-applies the last rollout with the new file |
| see which names a project needs | every `${NAME:?}` in `platform/deploy/<slug>/compose.yaml`; the Secrets page flags **required by compose, missing** for any without a value |
| see a value | open the secret → Reveal for 30 seconds, or Copy. Both are audited |
| go back to an earlier value | the secret's History tab → Restore |
| see what the VPS has | `sudo cut -d= -f1 /etc/graphite/<slug>.env` |

Names are paths, `/<project>/<NAME>`: the project is the compose folder, or `agent` for the deploy
agent, whose `/etc/graphite/agent.env` holds `DEPLOY_TOKEN`, `BIND` and `DEPLOY_REPORT_SECRET`.
A secret is `Secure` (masked in the list) or `Plain` (shown, for a public key). Deleting a name a
compose file still requires is refused. Every create, change, restore, delete and reveal writes an
audit row with the login.

The deploy agent refuses a rollout while a required name is missing, with `waiting on secrets: …`,
and changes nothing on the VPS.

## Keys

| Key | Where |
| :--- | :--- |
| `SECRETS_KEY` | your password manager and `/etc/graphite/console.env`; 32 random bytes, base64. Without it the store refuses to open |
| the console's own secrets | `/etc/graphite/console.env` only, placed by hand over SSH; the store refuses `/console/`. See [console.md](console.md#setting-it-up) |
| deploy token | your password manager, `/agent/DEPLOY_TOKEN` in the store, `DEPLOY_TOKEN` in `platform` (`production`) |
| `DEPLOY_REPORT_SECRET` | `/agent/`, `/graphite-hub/` and `console.env`; the same value in all three |
| Cloudflare API token | `CLOUDFLARE_API_TOKEN` in `platform` (`production`); Zone → DNS → Edit on `project-graphite.com` and Account → Cloudflare Pages → Edit |
| certificate DNS token | `/edge/CF_DNS_API_TOKEN` in the store, or the `secrets.env` the bootstrap wrote under `/srv/graphite/edge/` when `/etc/graphite/edge.env` does not exist |
| the Hub's GitHub OAuth App, webhook secret and SMTP | `/graphite-hub/`; see the Hub's README for the names |
| platform GitHub App | organisation secrets `PLATFORM_APP_CLIENT_ID` and `PLATFORM_APP_PRIVATE_KEY`, plus `/graphite-hub/` for write-back and `console.env` for releases from the console; see [rotating-the-deploy-app.md](rotating-the-deploy-app.md) |

`platform`'s `production` environment only runs from `main`, so a pull request branch cannot read
it. **Deploy** also reads the `platform` variables `DEPLOY_URL`
(`https://deploy.project-graphite.com`), `VPS_IPV4`, `CLOUDFLARE_ACCOUNT_ID` and
`CLOUDFLARE_ZONE_ID`.

## Creating the Cloudflare tokens

Cloudflare → My Profile → API Tokens → Create Token → Custom token, both limited to the
`project-graphite.com` zone:

| Token | Permissions | Goes in |
| :--- | :--- | :--- |
| platform | Zone → DNS → Edit; Account → Cloudflare Pages → Edit | `CLOUDFLARE_API_TOKEN` in `platform`'s `production` environment |
| certificate | Zone → Zone → Read; Zone → DNS → Edit | `/edge/CF_DNS_API_TOKEN` and the bootstrap command |

The account and zone IDs, on the domain's overview page, are the `CLOUDFLARE_ACCOUNT_ID` and
`CLOUDFLARE_ZONE_ID` variables in `platform`.

## Rotating

- **A project value:** edit it in the console, then Redeploy the project.
- **`SECRETS_KEY`:** generate a new key, set `SECRETS_KEY=<new>,<old>` in `console.env`, restart
  the console; it re-encrypts every version with the new key on start. Remove the old key from the
  file on the next restart, then destroy it.
- **The deploy token:** generate a new one, save it as `/agent/DEPLOY_TOKEN`, run
  `sudo systemctl restart graphite-agent` over SSH, then replace `DEPLOY_TOKEN` in `platform`.
- **`DEPLOY_REPORT_SECRET`:** change it in `/agent/`, `/graphite-hub/` and `console.env`, restart
  the console and redeploy the Hub.
- **The Hub's `SESSION_SECRET`:** change it and redeploy; every member is signed out and every
  subscriber link already sent stops working.
- **A Cloudflare token:** create the new one, replace `CLOUDFLARE_API_TOKEN` or
  `/edge/CF_DNS_API_TOKEN`, run **Deploy** with `edge` for the certificate token, then revoke the
  old one.

If `SECRETS_KEY` leaks together with the console's database or a backup of its `data` volume,
every value is readable: rotate the key and every project value.

## Break-glass

If the console is down, deploys still work from the files on the host. Over SSH, edit
`/etc/graphite/<slug>.env` as root (`NAME=value` lines, mode 600), then re-apply the project with
`sudo -u deploy /srv/graphite/bin/graphite-up <slug>`. When the console starts again it reads
every env file back into the store: a changed value becomes a new version, a new name a new
secret, a removed name is deleted, each with an audit row by `host`, so the store and the host
never disagree.
