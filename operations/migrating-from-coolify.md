# Moving off Coolify

A clean start: Coolify and everything it ran is removed, the VPS is wiped, and every project comes
back fresh from `platform`. No data is kept. Sites are down from step 4 until step 7 finishes.

## 1. Save the outside credentials

From each Coolify application's Environment Variables, copy the values other services issued:
the Gmail app password, `TMDB_READ_ACCESS_TOKEN`, `IGDB_CLIENT_ID`, `IGDB_CLIENT_SECRET`,
`RAWG_API_KEY`, `VAPID_*`, and `GAME_SOURCE` or `DISABLED_SOURCES` if set. Everything else gets a new
value.

## 2. Merge the pull requests

In this order:

1. `actions`: *record static-site builds*, then *remove the Coolify webhook*.
2. `platform`: *deploy every project from platform*. Every project starts `enabled: false`.
3. `graphite-message-lab`, `graphite-tracker`, `strata`, `pixelpocket`: *register only the host*, then
   *retire the Coolify production compose*.
4. `secrets`, `graphite-hub`, `digital-to-physical`, `project-template`, `.github`, `docs`.
5. `console`.

The project and site merges record fresh builds in `platform/tags/`.

## 3. Create the credentials

1. **Deploy token.** `openssl rand -hex 32`. It goes to the VPS, `platform` and `secrets`.
2. **age key.** On your machine:

   ```powershell
   New-Item -ItemType Directory -Force "$env:AppData\sops\age" | Out-Null
   age-keygen -o "$env:AppData\sops\age\keys.txt"
   ```

   It prints the public key (`age1…`). Keep the file in your password manager.
3. **Cloudflare API tokens.** My Profile → API Tokens → two custom tokens:
   - for `platform`: Zone → DNS → Edit on `project-graphite.com` and Account → Cloudflare Pages →
     Edit;
   - for the VPS's certificates: Zone → DNS → Edit on `project-graphite.com` only.

   Copy the account ID and zone ID too.
4. **Console sign-in.**
   - A GitHub OAuth App in the organisation, `Graphite Console`, callback
     `https://console.project-graphite.com/auth/callback`.
   - Zero Trust → Settings → Authentication → add GitHub, using a second OAuth App with callback
     `https://<team>.cloudflareaccess.com/cdn-cgi/access/callback`.
   - Zero Trust → Access → Applications → Self-hosted `console.project-graphite.com`, GitHub
     login, Allow the organisation `Project-Graphite`. Copy its AUD tag.
   - Cloudflare → SSL/TLS: set the mode to **Full (strict)**, which the proxied console needs.

## 4. Wipe the VPS

Delete `COOLIFY_WEBHOOK` and `COOLIFY_TOKEN` from the four project repositories, then:

```sh
ssh -i ~/.ssh/graphite-ovh ubuntu@135.125.254.59
sudo docker ps -aq | xargs -r sudo docker rm -f
sudo docker volume ls -q | xargs -r sudo docker volume rm
sudo docker network prune -f
sudo docker image prune -af
sudo rm -rf /data/coolify
sudo ss -tlnp
```

The last command must show nothing on 80, 443 or 8000. Docker stays installed.

## 5. Bootstrap

```sh
curl -fsSLO https://raw.githubusercontent.com/project-graphite/platform/main/host/bootstrap.sh
sudo DEPLOY_TOKEN='<deploy token>' CF_DNS_API_TOKEN='<certificate token>' sh bootstrap.sh
```

## 6. Configure GitHub and the secrets

In `platform`, Settings → Environments → `production`, limited to `main`, then:

```powershell
$repo = "Project-Graphite/platform"
gh secret set DEPLOY_TOKEN --repo $repo --env production
gh secret set CLOUDFLARE_API_TOKEN --repo $repo --env production
gh variable set DEPLOY_URL --repo $repo --body https://deploy.project-graphite.com
gh variable set VPS_IPV4 --repo $repo --body 135.125.254.59
gh variable set CLOUDFLARE_ACCOUNT_ID --repo $repo --body <account id>
gh variable set CLOUDFLARE_ZONE_ID --repo $repo --body <zone id>
```

In `secrets`: the secrets `DEPLOY_TOKEN` and `SOPS_AGE_KEY` (the private key), the variable
`DEPLOY_URL`, and your public key in place of `AGE_PUBLIC_KEY` in `.sops.yaml`.

Then run platform **Deploy** with `edge`: it points `deploy.project-graphite.com` at the VPS and
re-applies the edge, which proves the token and the agent work.

Now write one file per project with `sops edit <name>.env`, using `openssl rand -base64 32` for new
values:

| File | Required | Optional |
| :--- | :--- | :--- |
| `edge.env` | `CF_DNS_API_TOKEN` (the certificate token) | |
| `graphite-message-lab.env` | `SECRET_KEY`, `POSTGRES_PASSWORD` | |
| `graphite-tracker.env` | `POSTGRES_PASSWORD`, `AUTH_ACCESS_TOKEN_SECRET`, `EMAIL_HOST`, `DEFAULT_FROM_EMAIL` | `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `TMDB_READ_ACCESS_TOKEN`, `GAME_SOURCE`, `IGDB_CLIENT_ID`, `IGDB_CLIENT_SECRET`, `RAWG_API_KEY`, `DISABLED_SOURCES` |
| `strata.env` | `POSTGRES_PASSWORD`, `REDIS_PASSWORD`, `AUTH_ACCESS_TOKEN_SECRET`, `DATA_ENCRYPTION_KEY` (32 bytes), `EMAIL_HOST`, `DEFAULT_FROM_EMAIL` | `EMAIL_PORT`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `VAPID_*`, `BACKUP_KEEP_DAYS` |
| `pixelpocket.env` | `MONGO_PASSWORD`, `MINIO_ROOT_PASSWORD`, `S3_SECRET_KEY`, `JWT_SECRET`, `EMAIL_HOST`, `EMAIL_FROM` | `EMAIL_PORT`, `EMAIL_USERNAME`, `EMAIL_PASSWORD` |
| `console.env` | `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `SESSION_SECRET`, `ACCESS_TEAM`, `ACCESS_AUD` | |

Push; **Apply** sends each file to the agent.

## 7. Deploy everything

1. Cloudflare → Workers & Pages: delete the Git-connected `graphite-hub` and `digital-to-physical`
   projects.
2. In `platform`, merge one pull request setting `enabled: true` in `deploy/` for
   `graphite-message-lab`, `graphite-tracker`, `strata`, `pixelpocket`, `console`, `graphite-hub` and
   `digital-to-physical`. **Deploy** starts every project, publishes its route, points its
   hostname at the VPS, and builds and uploads both sites.
3. Register on Graphite Tracker and grant yourself the system manager from the console's **Shell**:
   `docker exec -it graphite-tracker-app-1 npm run system-manager:grant --workspace server -- <email>`.

## 8. Finish

- [ ] every site answers, and the console shows every container
- [ ] Message Lab sleeps after 30 minutes and wakes on the next visit
- [ ] the `coolify.project-graphite.com` DNS record is deleted
- [ ] the OVH firewall still allows only 22 from your IP, 80, 443/tcp and 443/udp
- [ ] after a day, each service's `reserve` matches what the console shows
