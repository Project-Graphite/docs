# Deploying Graphite Tracker

This is the procedure used to deploy Graphite Tracker to the shared OVH VPS through Coolify, from
signing in to the server through automatic redeployment on every push to `main`. It starts where
[ovh-coolify.md](ovh-coolify.md) sections 1 to 10 end: the VPS, edge firewall and Coolify are
already running.

Only the capacity check, the image check and the administrator setup use the VPS terminal. Every
other step happens in GitHub, Cloudflare or Coolify.

## Deployment

| Item | Value |
| :--- | :--- |
| Repository | `Project-Graphite/graphite-tracker` |
| Public URL | `https://graphite-tracker.project-graphite.com` |
| Image | `ghcr.io/project-graphite/graphite-tracker/app` |
| Image tags | `sha-<first 7 characters of the commit>` and `latest` |
| Compose file | `compose.production.yaml` |
| Services | `server` (public, internal port 3000), `postgres`, `redis` |
| Coolify `IMAGE_TAG` | `latest` |
| Email sender | `Graphite Tracker <projectgraphite.org@gmail.com>` through Gmail SMTP |

## 1. Sign in to the VPS

From Windows PowerShell:

~~~powershell
ssh -i "$env:USERPROFILE\.ssh\graphite-ovh" ubuntu@135.125.254.59
~~~

The prompt becomes `ubuntu@vps-2283122d`.

## 2. Check capacity

~~~sh
free -h && df -h / && sudo docker stats --no-stream
~~~

The tracker's container limits add 992 MiB: 512 MiB `server`, 384 MiB `postgres` and 96 MiB
`redis`. Limits are ceilings, not usage. Stop and review capacity if steady used memory is already
above about 1.8 GiB, because adding the full 992 MiB would then pass the 75% review threshold in
[ovh-coolify.md](ovh-coolify.md) section 19. Free disk must stay well above 10 GB.

Baseline before the first deployment, 2026-09-26:

| Measure | Value |
| :--- | :--- |
| Memory | 1.2 GiB used of 3.7 GiB, 2.6 GiB available |
| Swap | 0 B used of 2 GiB |
| Disk | 11 GB used, 28 GB free of 38 GB |
| Coolify containers | about 705 MiB together |
| Message Lab containers | about 295 MiB together |

Message Lab's comparable services suggest the tracker adds about 250 MiB in practice, for roughly
1.45 GiB (39%). At its full limits the host would reach about 2.2 GiB (59%).

## 3. Watch usage without commands

Enable Coolify's metrics before the first deployment so the charts show the change:

1. Open **Servers** and select this server.
2. Open **Configuration → Metrics**. Some Coolify versions label it **Sentinel → Configuration**.
3. Select **Enable Metrics** and **Save**.

The `coolify-sentinel` container already runs at about 18 MiB, so metrics add almost nothing. The
defaults sample every 10 seconds and keep 7 days.

Coolify then charts whole-server CPU and memory. It does not chart Docker Compose applications, so
neither Message Lab nor the tracker gets per-application charts, and there is no disk chart yet.
Free disk and per-container memory still need `df -h /` and `sudo docker stats --no-stream`.

The OVH Control Panel also graphs the VPS's CPU, RAM and network, but its RAM figure includes the
page cache and can read close to 100% on a healthy host. Use Coolify's memory chart for capacity
decisions.

Do not add Netdata, Grafana or another monitoring stack; [ovh-coolify.md](ovh-coolify.md)
section 18 rules them out on this host.

## 4. Publish the image from GitHub Actions

In `.github/workflows/main.yml`, the `images` job calls the shared build workflow once the checks
pass:

~~~yaml
  images:
    name: Images
    needs: [services, checks]
    permissions:
      contents: read
      packages: write
    uses: project-graphite/actions/.github/workflows/reusable-docker-build-push.yml@main
    with:
      services: ${{ needs.services.outputs.services }}
~~~

The Detect job returns `["app"]`, so this publishes
`ghcr.io/project-graphite/graphite-tracker/app` tagged `sha-<first 7 characters>` and `latest`.
The last stage of `app/Dockerfile` is `server`, so the build needs no target. Push to `main` and
wait for the **Main** run to pass.

## 5. Make the image pullable

A newly published package can start private. In GitHub, open the Project-Graphite organisation →
**Packages** → `graphite-tracker/app` → **Package settings** → **Change visibility** and make it
**Public**, as Message Lab's packages are. A private package would instead need a narrowly scoped
registry credential in Coolify.

Verify from the VPS:

~~~sh
sudo docker manifest inspect ghcr.io/project-graphite/graphite-tracker/app:latest
~~~

## 6. Create the DNS record

In Cloudflare, create:

| Type | Name | Content | TTL | Proxy status |
| :--- | :--- | :--- | :--- | :--- |
| A | `graphite-tracker` | `135.125.254.59` | 300 | DNS only |

The name must be exactly `graphite-tracker`. `compose.production.yaml` fixes
`AUTH_TRUSTED_ORIGINS` to `https://graphite-tracker.project-graphite.com`, and the authentication
routes reject requests from any other origin.

## 7. Create the Coolify application

1. Open **Projects → Graphite → production**.
2. Select **+ New → Public Repository**.
3. Enter `https://github.com/Project-Graphite/graphite-tracker` and select branch `main`.
4. Choose **Docker Compose** and set the Compose file to `/compose.production.yaml`.
5. Save and select **Reload Compose**.

The parsed services must be `server`, `postgres` and `redis`, with two persistent storages for the
PostgreSQL and Redis volumes.

Keep the application's **Auto Deploy** switch off. It fires on the Git push itself, before GitHub
Actions has published the new image, so it would redeploy the previous `latest`. Section 12 lets
GitHub Actions trigger the deployment instead.

## 8. Configure environment variables

Under **Environment Variables**, set:

| Name | Value | Required |
| :--- | :--- | :--- |
| `IMAGE_TAG` | `latest` | yes |
| `POSTGRES_PASSWORD` | unique generated secret | yes |
| `AUTH_ACCESS_TOKEN_SECRET` | separate long generated secret | yes |
| `TMDB_READ_ACCESS_TOKEN` | TMDB read access token | no |
| `GAME_SOURCE` | `igdb` or `rawg`; defaults to `igdb` | no |
| `IGDB_CLIENT_ID`, `IGDB_CLIENT_SECRET` | IGDB credentials, when `GAME_SOURCE` is `igdb` | no |
| `RAWG_API_KEY` | RAWG key, when `GAME_SOURCE` is `rawg` | no |
| `EMAIL_HOST` | `smtp.gmail.com` | yes |
| `EMAIL_PORT` | `587`; `465` uses TLS from the start | no, defaults to `587` |
| `EMAIL_HOST_USER` | `projectgraphite.org@gmail.com` | yes, for Gmail |
| `EMAIL_HOST_PASSWORD` | Gmail app password without spaces | yes, for Gmail |
| `DEFAULT_FROM_EMAIL` | `Graphite Tracker <projectgraphite.org@gmail.com>` | yes |

`IMAGE_TAG` is `latest`, not `main` as for Message Lab, because the shared build workflow tags
`latest`. Compose refuses to start without `POSTGRES_PASSWORD`, `AUTH_ACCESS_TOKEN_SECRET`,
`EMAIL_HOST` and `DEFAULT_FROM_EMAIL`; a deployment attempted before they exist fails and the
running version stays up. An empty source credential only turns that connector off. On any port
other than `465`, a configured `EMAIL_HOST_USER` requires the server to upgrade the connection with
STARTTLS.

`AUTH_ACCESS_TOKEN_SECRET` also signs the one-click unsubscribe links in digest emails. Rotating it
signs everyone out and invalidates every unsubscribe link already sent.

Generate secrets in a password manager and keep them only in Coolify. Never place them in GitHub,
Compose, screenshots, shell history, documentation or chat.

### Gmail sender

`projectgraphite.org@gmail.com` is the shared Project Graphite sender. Every project sends from the
same address under its own display name, such as `Graphite Tracker`.

Gmail SMTP needs an app password; the account's normal password does not work:

1. Sign in to Google as `projectgraphite.org@gmail.com`.
2. Open **Google Account → Security → 2-Step Verification** and turn it on with a phone or Google
   prompt. An account protected only by security keys cannot create app passwords.
3. Open [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords). Google
   often hides this page from its menus.
4. Name the password after the project, for example `graphite-tracker`, and select **Create**.
5. Google shows 16 characters once, in groups of four such as `abcd efgh ijkl mnop`. Enter them in
   `EMAIL_HOST_PASSWORD` without spaces, as `abcdefghijklmnop`, and keep a copy in the password
   manager.

If an app password is ever exposed, delete it on the same page and create another. The page is
unavailable while 2-Step Verification is off, for security-key-only accounts, and under Advanced
Protection.

Mail comes from a `gmail.com` address, so `project-graphite.com` needs no SPF, DKIM or DMARC
records. Gmail rewrites a `DEFAULT_FROM_EMAIL` on another domain to the account's own address.

The account sends at most about 500 messages in any rolling 24 hours, shared by every project that
uses it. Each verification, password reset and digest counts. Move to a dedicated email provider
before digests approach that limit.

## 9. Set the public domain

Give only the `server` service a domain, with the container port appended to the URL:

~~~text
https://graphite-tracker.project-graphite.com:3000
~~~

The `:3000` suffix tells Coolify's proxy which container port receives the traffic. Visitors still
connect on HTTPS 443. There is no separate port field for this; `compose.production.yaml` already
declares `expose: '3000'`. Leave any host port mapping empty, because a mapping publishes the port
directly on the VPS and bypasses the proxy.

If Coolify reports:

~~~text
Invalid URL: https://https://graphite-tracker.project-graphite.com:3000. The hostname must be a fully qualified domain name.
~~~

the field already contained `https://` when the value was pasted. Clear the field and enter the
value once. If the field shows `https://` as a fixed prefix, type only
`graphite-tracker.project-graphite.com:3000`.

`postgres` and `redis` receive no domain.

## 10. Deploy for the first time

Select **Deploy**. On every start the `server` container runs `prisma migrate deploy` before the
application, so the first deployment applies every migration to the new, empty database. Wait for
all three services to become healthy and for Coolify to obtain the Let's Encrypt certificate.

Verify:

~~~sh
curl -I https://graphite-tracker.project-graphite.com
curl https://graphite-tracker.project-graphite.com/api/v1/health
~~~

The site must return HTTP 200 with a valid certificate.

## 11. Create the administrator

The administrator is granted once, operationally; the application has no promotion endpoint.

### Register and verify

Register at `https://graphite-tracker.project-graphite.com/register` and open the verification
link emailed from `DEFAULT_FROM_EMAIL`. This is also the first test of real delivery: the message
must reach the inbox, not spam, and link to `https://graphite-tracker.project-graphite.com`. Test
**Forgot your password?** on the sign-in page as well.

If no message arrives, check spam, then the `server` logs in Coolify. An SMTP authentication error
means the app password is wrong or 2-Step Verification is off. An unverified account can request a
new link from the registration confirmation or after a refused sign-in.

### Find the containers

~~~sh
sudo docker ps --format 'table {{.Names}}\t{{.Image}}'
~~~

Coolify names each container `<service>-<resource ID>-<timestamp>`. The tracker's `server`
container runs `ghcr.io/project-graphite/graphite-tracker/app`; its PostgreSQL container is the
`postgres-...` entry with the same resource ID. Do not use Message Lab's PostgreSQL container.

### Grant the role

~~~sh
sudo docker exec -it SERVER_CONTAINER npm run admin:grant --workspace server -- ADMIN_EMAIL
~~~

The command prints `<handle> is now the administrator`. Its errors mean:

| Error | Cause |
| :--- | :--- |
| `No account uses that email address` | The account is not registered in production, or the address differs |
| `The account must be verified and active` | The verification link has not been opened |
| `An administrator already exists` | The one-time grant is already done |

### Verify an account without email

Only when email cannot be delivered, mark the administrator's account verified directly. This
writes one row in the production database. The first administrator was verified this way before
email delivery existed.

~~~sh
sudo docker exec -it POSTGRES_CONTAINER psql -U graphite -d graphite_tracker -c "UPDATE users SET verified_at = now() WHERE email = 'ADMIN_EMAIL' AND verified_at IS NULL RETURNING handle, verified_at;"
~~~

`UPDATE 1` with the handle confirms the change. `UPDATE 0` means no unverified account uses that
email address.

## 12. Enable automatic deployments

After this section, every push to `main` deploys itself:

~~~text
push to main
  → Checks   lint, tests and build
  → Images   publishes app:sha-<7 characters> and app:latest to GHCR
  → Deploy   sends an authenticated POST to the tracker's Coolify webhook
  → Coolify  pulls app:latest, starts the new server (migrations first),
             waits for its health check and replaces the old container
~~~

Create the token, webhook and GitHub secrets before pushing the workflow change. Otherwise the
first run's Deploy job fails on an empty webhook secret.

### Create a deploy-only token

1. Confirm **Settings → Configuration → Advanced → API Access** is enabled. It was enabled for
   Message Lab.
2. Open **Keys & Tokens → API Tokens** and select **Add**.
3. Name it for the tracker's GitHub deployment, for example `github-graphite-tracker`.
4. Grant only `deploy`. A leaked token can then only trigger deployments; it cannot read secrets
   or change settings.
5. Choose a finite expiration, for example one year.
6. Create it and copy it once into a password manager. Copy the whole value, including the numeric
   ID and `|` at the start.

Use a token separate from Message Lab's so either can be revoked without breaking the other. When
the token expires, the Deploy job fails with HTTP 401; create a replacement and update the GitHub
secret.

### Copy the resource webhook

Open the Graphite Tracker application → **Webhooks** and copy **Deploy Webhook (auth required)**.
It is a URL on `coolify.project-graphite.com` that targets only this application. Ignore the
Git-provider webhook fields on the same page; they serve Coolify's own Git-triggered deployments.

### Store GitHub secrets

In the graphite-tracker repository, open **Settings → Secrets and variables → Actions**, stay on
the **Secrets** tab and create:

| Secret | Value |
| :--- | :--- |
| `COOLIFY_WEBHOOK` | the Deploy Webhook URL |
| `COOLIFY_TOKEN` | the deploy-only token |

The names must match exactly because the workflow reads them by name. Use secrets, not variables:
secrets are masked in logs and cannot be read back.

### Add the Deploy job

In `.github/workflows/main.yml`, add after the `images` job:

~~~yaml
  deploy:
    name: Deploy to Coolify
    needs: images
    runs-on: ubuntu-latest
    steps:
      - name: Trigger deployment
        env:
          COOLIFY_TOKEN: ${{ secrets.COOLIFY_TOKEN }}
          COOLIFY_WEBHOOK: ${{ secrets.COOLIFY_WEBHOOK }}
        run: |
          curl --fail --show-error --silent --request POST "$COOLIFY_WEBHOOK" \
            --header "Authorization: Bearer $COOLIFY_TOKEN"
~~~

`needs: images` means nothing deploys unless the checks pass and the image is published. The
request must be `POST`; this Coolify installation returns HTTP 405 for `GET`. `--fail` turns the
job red when Coolify rejects the call.

### Register the deployment

In the same commit, set the `deploy` block in `.graphite.yml`:

~~~yaml
    deploy:
      kind: compose
      host: graphite-tracker.project-graphite.com
      source:
        file: compose.production.yaml
        service: server
        port: 3000
~~~

This records the deployment in the platform registry; Coolify does not read it. The tracker has no
`registry.yml` workflow, so the platform's daily **Sync registry** run at 05:17 UTC picks up the
change, or it can be run manually from the `platform` repository's **Actions** tab. `listed`
decides whether the project appears on the Hub, and the Hub links to the live site only when
`status` is `live`.

### Push and verify

The push that adds the Deploy job is also its first test:

1. In GitHub **Actions**, the **Main** run must pass Checks, Images and **Deploy to Coolify**. A
   green Deploy job only means Coolify queued the deployment.
2. In Coolify, the application's **Deployments** list must show a new deployment within seconds.
   Wait for it to finish and for the services to become healthy.
3. The site must load normally.

When the Deploy job fails:

| Symptom | Cause |
| :--- | :--- |
| HTTP 401 | The token is wrong, expired or missing its `N\|` prefix |
| HTTP 404 | The webhook URL is wrong |
| curl reports no URL | A secret name does not match |

Every start runs pending migrations, so a merge to `main` that adds a Prisma migration applies it to
production on the next deployment. Review migrations before merging.

## 13. Roll back

1. Identify the last known-good `sha-...` tag, the first seven characters of its commit.
2. Check whether a migration prevents running the older image; rolling back the image does not
   roll back the database.
3. Change Coolify's `IMAGE_TAG` from `latest` to that tag.
4. Select **Deploy**, not Restart.
5. Verify health and the public site.
6. Set `IMAGE_TAG` back to `latest` and deploy once to resume automatic updates.

Never roll back a database by deleting volumes.

## 14. Back up the database

The Coolify instance backup does not cover the tracker's PostgreSQL volume. Set up a PostgreSQL
backup and complete a restore test before inviting real users.

## References

- [Coolify Sentinel and metrics](https://coolify.io/docs/knowledge-base/server/sentinel)
- [Coolify metrics](https://coolify.io/docs/core/observability/monitoring/metrics)
- [Coolify Docker Compose applications](https://coolify.io/docs/applications/builds/docker-compose)
- [Coolify API authorization](https://coolify.io/docs/api-reference/authorization)
- [Coolify deployment webhooks](https://coolify.io/docs/core/automation/deploy-webhooks)
- [Google app passwords](https://support.google.com/accounts/answer/185833?hl=en)
- [Gmail sending limits](https://support.google.com/mail/answer/22839?hl=en)
