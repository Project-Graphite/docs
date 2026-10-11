# Migrating from Coolify

Moves every project from Coolify to platform deploys, one at a time, on the same VPS. Coolify keeps
serving a project until its own cutover, so nothing changes for visitors until then.

## 1. Prepare

Do these once, in order.

### Merge the pull requests

1. `actions`: *record static-site builds and add the vps-ssh action*
2. `platform`: *deploy every project from platform*
3. `graphite-message-lab`, `graphite-tracker`, `strata`, `pixelpocket`: *register only the host*
4. `secrets`: *apply SOPS-encrypted project secrets*
5. `graphite-hub`, `digital-to-physical`, `project-template`, `.github`, `docs`

Every project starts with `enabled: false`, so merging deploys nothing. Leave each project's
*retire the Coolify production compose* pull request open until its cutover.

### Create the keys on your machine

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\graphite-deploy" -C graphite-deploy -N '""'
New-Item -ItemType Directory -Force "$env:AppData\sops\age" | Out-Null
age-keygen -o "$env:AppData\sops\age\keys.txt"
```

`age-keygen` prints the public key (`age1…`). Store the private key file's contents in your password
manager. Install [sops](https://github.com/getsops/sops/releases) and
[age](https://github.com/FiloSottile/age/releases) first if they are missing.

### Set up Cloudflare

1. **Tunnel.** Zero Trust → Networks → Tunnels → Create a tunnel → Cloudflared, named `graphite`.
   Copy the token from the install command and the tunnel ID. Do not add public hostnames; platform
   writes the tunnel's routes.
2. **Service token.** Zero Trust → Access → Service credentials → Create service token
   `graphite-deploy`. Copy the client ID and secret.
3. **SSH application.** Zero Trust → Access → Applications → Add → Self-hosted, domain
   `ssh.project-graphite.com`, one policy with action **Service Auth** that includes the
   `graphite-deploy` service token.
4. **API token.** My Profile → API Tokens → Create a custom token with Zone → DNS → Edit on
   `project-graphite.com`, Account → Cloudflare Pages → Edit, and Account → Cloudflare Tunnel →
   Edit.
5. Copy the account ID and zone ID from the domain's overview page.

### Bootstrap the VPS

```sh
ssh -i ~/.ssh/graphite-ovh ubuntu@135.125.254.59
curl -fsSLO https://raw.githubusercontent.com/project-graphite/platform/main/host/bootstrap.sh
sudo TUNNEL_TOKEN='<tunnel token>' sh bootstrap.sh '<contents of graphite-deploy.pub>'
cat /etc/ssh/ssh_host_ed25519_key.pub
```

The last line is the host key. Your known-hosts value is `ssh.project-graphite.com ` followed by
it.

### Configure GitHub

In `platform`, Settings → Environments → New environment `production`, with deployment branches
limited to `main`. Then, from PowerShell in the folder holding the files:

```powershell
$repo = "Project-Graphite/platform"
Get-Content "$env:USERPROFILE\.ssh\graphite-deploy" -Raw | gh secret set DEPLOY_SSH_KEY --repo $repo --env production
gh secret set DEPLOY_KNOWN_HOSTS --repo $repo --env production
gh secret set CF_ACCESS_CLIENT_ID --repo $repo --env production
gh secret set CF_ACCESS_CLIENT_SECRET --repo $repo --env production
gh secret set CLOUDFLARE_API_TOKEN --repo $repo --env production
gh variable set DEPLOY_SSH_HOST --repo $repo --body ssh.project-graphite.com
gh variable set CLOUDFLARE_ACCOUNT_ID --repo $repo --body <account id>
gh variable set CLOUDFLARE_ZONE_ID --repo $repo --body <zone id>
gh variable set TUNNEL_ID --repo $repo --body <tunnel id>
```

`gh secret set` without `--body` prompts for the value, so nothing lands in your shell history.

In `secrets`, set the same `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS`, `CF_ACCESS_CLIENT_ID` and
`CF_ACCESS_CLIENT_SECRET` as repository secrets, `SOPS_AGE_KEY` to the private age key, and the
variable `DEPLOY_SSH_HOST`. Then replace `AGE_PUBLIC_KEY` in its `.sops.yaml` with your public key,
and add the tunnel token:

```sh
sops edit edge.env          # TUNNEL_TOKEN=<tunnel token>
git add .sops.yaml edge.env
git commit -m "chore: add the age key and the edge secrets"
git push
```

### Start the edge from platform

Actions → **Deploy** → Run workflow with `edge`. It writes the tunnel routes, points
`ssh.project-graphite.com` at the tunnel, and re-applies the edge over SSH. A green run proves SSH,
Access and the API token all work.

## 2. Move a project

Do Message Lab first; it is the test application. Then `pixelpocket`, `graphite-tracker`, and
`strata` last.

1. **Copy its environment.** In Coolify, open the application → Environment Variables and copy every
   variable except `IMAGE_TAG`. In `secrets`:

   ```sh
   sops edit <slug>.env        # paste NAME=value lines
   git add <slug>.env && git commit -m "chore(<slug>): add secrets" && git push
   ```

2. **Stop it in Coolify.** The application → Stop. The site is down from here until step 4.

3. **Copy its volumes.** On the VPS, list the Coolify volumes (named `<application id>_<volume>`),
   then copy each one:

   ```sh
   sudo docker volume ls
   curl -fsSLO https://raw.githubusercontent.com/project-graphite/platform/main/host/migrate-volume.sh
   sudo sh migrate-volume.sh <coolify volume> <slug> <volume>
   ```

   | Project | Volumes |
   | :--- | :--- |
   | `graphite-message-lab` | `postgres-data`, `redis-data` |
   | `graphite-tracker` | `postgres-data`, `redis-data` |
   | `pixelpocket` | `mongo-data`, `minio-data` |
   | `strata` | `postgres-data`, `redis-data`, `files-data`, `backups` |

4. **Enable it.** Open a pull request in `platform` setting `enabled: true` in
   `deploy/<slug>/values.yaml`, and merge it. **Deploy** checks the secrets exist, starts the
   project, publishes its route, and replaces the hostname's A record with the tunnel.

5. **Verify.** Open the site, sign in, check data is there, and read
   `sudo docker compose -p <slug> logs --tail 100`.

6. **Stop Coolify deploys.** Delete `COOLIFY_WEBHOOK` and `COOLIFY_TOKEN` from the project
   repository's Actions secrets, then merge its *retire the Coolify production compose* pull
   request.

7. **Delete it in Coolify** once you are satisfied, then remove the old volumes:
   `sudo docker volume rm <coolify volume>`.

**Rolling back during a cutover:** set `enabled: false` (its deploy stops the project), start the
application again in Coolify, and recreate the hostname's DNS-only A record to `135.125.254.59`.

## 3. Move the static sites

For `graphite-hub` and `digital-to-physical`:

1. Push to the repository's `main`, or rerun its **Main** workflow, so `tags/<slug>.json` records a
   build.
2. In Cloudflare → Workers & Pages, find the site's Git-connected project. If its name equals the
   slug, delete it; otherwise remove its custom domain. The site is down until step 3 finishes.
3. Set `enabled: true` in `platform/deploy/<slug>/values.yaml` and merge. **Deploy** builds the
   recorded commit, creates the direct-upload Pages project, attaches the domain and points DNS at
   it.
4. Verify the site, then delete any old project you kept.

`traceboard` stays on Git integration: it is private, so organisation secrets cannot reach it to
record builds.

## 4. Remove Coolify

After every project is moved:

```sh
sudo docker ps -a --filter name=coolify --format '{{.Names}}'
sudo docker rm -f $(sudo docker ps -aq --filter name=coolify)
sudo docker volume ls --filter name=coolify
sudo docker volume rm coolify-db coolify-redis
sudo docker network rm coolify
sudo rm -rf /data/coolify
```

Then delete the `coolify.project-graphite.com` DNS record, remove the TCP 80, TCP 443 and UDP 443
rules from the OVH edge firewall, and merge the `actions` pull request *remove the Coolify webhook*.
Check `gh secret list` in each project repository shows no `COOLIFY_*` left.

## 5. Done when

- [ ] every compose project shows `enabled: true` and answers on its hostname
- [ ] Message Lab sleeps after 30 minutes and wakes on the next visit
- [ ] the Hub and Digital to Physical deploy from platform
- [ ] no `COOLIFY_*` secrets, no Coolify containers, no open web ports
- [ ] `docker stats` matches `deploy/host.yaml`; update each `reserve` to measured use
