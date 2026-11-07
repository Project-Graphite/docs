# The console

`https://console.project-graphite.com` is the one place to look at and operate the VPS. Code:
[`console`](https://github.com/project-graphite/console). Production:
`platform/deploy/console/`.

| Page | Shows |
| :--- | :--- |
| Overview | host CPU, memory, swap and disk; host usage charts; storage by images, volumes, container layers and build cache; one row per project with status, CPU, memory, storage and a **needs redeploy** hint, filterable and pinnable; the last backup run |
| Project | usage chart with a line per container; containers; images and volumes with sizes; what changed since the last rollout and a Redeploy button; a release control that writes `IMAGE_TAG` or `ENABLED` into `platform` through the platform App; the project's secrets, for those granted them |
| Container | usage chart, details and networks, live logs; start, stop, restart, a shell and its ports |
| Logs | the logs the console collects from every container, kept 7 days, so they survive a redeploy |
| Images | every image, its size and users; remove dangling images |
| Volumes | every volume, its project, size and whether it is in use |
| Shell | a root shell on the VPS |
| Alerts | what needs attention now and what resolved in the last 7 days, with the rules and their thresholds; see [monitoring.md](monitoring.md) |
| Secrets | owners only: every project's secrets; see below |
| Backups | owners only: every archive, the schedule, run, download, import and restore; see below |
| Admin | owners only: every organisation member, when they last signed in, and their access |
| Docs | the handbook, rendered from the `docs` repository |

Every organisation member can read every page except Secrets, Backups and Admin. Changing anything
needs access, which an owner sets per member on the Admin page:

| Access | Lets them |
| :--- | :--- |
| a project | start, stop and restart it, redeploy it, and open its containers' terminals and ports |
| a project's secrets | see and change that project's secrets, on its Project page |
| images | remove dangling images |
| host shell | open the root shell on the VPS |
| everything | all of the above for every project, except secrets, which are granted per project |

Owners have everything.

Usage charts cover the last hour, 5 hours, day, 7 days or 30 days. Hover a chart for the time and
every line's value; the line nearest the pointer is highlighted. Pins and the chosen range are kept
in your browser.

## Secrets

The store behind the page is described in [managing-secrets.md](managing-secrets.md). The page is
one list of `/<project>/<NAME>` rows with type, description, version, last change and who made it,
filtered by a path prefix or text, with a filter for projects missing a secret. Create opens a
sheet with Create and Import tabs; a row opens the secret with Overview (Reveal for 30 seconds,
Copy), History (every version, Restore) and Usage (the lines of the applied compose file that
reference it). A project header shows **needs redeploy** when a value changed after its last
rollout, with the Redeploy button, and **required by compose, missing** for a `${NAME:?}` without
a value. Delete is confirmed and refused for a name the compose file still requires.

Redeploy runs `graphite-up` on the host as a transient systemd unit and shows its log; the
rollout is reported to the Hub like any other.

## Backups

The service behind the page is described in the
[platform README](https://github.com/project-graphite/platform#the-edge-and-backups). The page
shows the schedule (on or off, every 1 to 168 hours from a chosen UTC hour, kept 1 to 90 days;
nightly at 03:00 UTC and 14 days by default), how many projects' latest archives verified, and per
project every archive with its date, size, items and **verified** or **failed** with the reason,
a Download that streams the tar to your machine, a Restore, and **Back up now**. Import uploads a
tar for a project and verifies it on arrival. Restore stops the project, verifies the archive
again, restores the databases and volumes, starts it and waits for health; the console refuses an
archive whose manifest is not that project's. Every download, run, import, restore and schedule
change is written to `audit.log` beside the archives. A project with no archive in 26 hours, or
whose last one failed, raises an alert.

## Ports

A container's web port opens at `/open/<container>/<port>/`, for example MinIO's console. Apps that
build absolute links from `/` may not render there; use an SSH tunnel for those. For any TCP port,
the container page shows the forwarding command:

```sh
ssh -L 5432:<container ip>:5432 ubuntu@135.125.254.59
```

## Sign-in

The console's host is proxied through Cloudflare, and Cloudflare Access admits only organisation
members. The console then signs in with GitHub: an organisation admin is an owner, and anyone
else is a member. Sessions last 1 hour. Every change rechecks the role with GitHub and the
member's access before it runs, so a demoted owner or a removed grant takes effect within a minute.
Removing access on the Admin page also closes that member's open terminals for it at once.

## Setting it up

Once, before the console's first deploy. The values in the last column go in
`/etc/graphite/console.env` on the VPS, placed by hand over SSH as root with mode 600; the
console's own secrets are the one file the store does not manage.

1. **GitHub OAuth App for the console.** Organisation settings → Developer settings → OAuth Apps →
   New. Homepage `https://console.project-graphite.com`, callback
   `https://console.project-graphite.com/auth/callback`. Its client ID and a new client secret are
   `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET`.
2. **GitHub OAuth App for Access.** A second app with homepage
   `https://project-graphite.cloudflareaccess.com` and callback
   `https://project-graphite.cloudflareaccess.com/cdn-cgi/access/callback`.
3. **Login method.** Cloudflare Zero Trust (Free plan) → Integrations → Identity providers → Add →
   GitHub, with the second app's client ID and secret. On GitHub's authorisation page, **Grant**
   Project-Graphite before **Authorize**, or the organisation rule below matches no one. The team
   name, `project-graphite`, is `ACCESS_TEAM`.
4. **Access application.** Zero Trust → Access controls → Applications → Create → Self-hosted,
   public hostname `console.project-graphite.com`, an Allow policy that includes the GitHub
   organisation `Project-Graphite`, GitHub as the only identity provider. Its Application Audience
   (AUD) tag, under Configure → Additional settings, is `ACCESS_AUD`.
5. **SSL mode.** The `project-graphite.com` zone's SSL/TLS mode is Full (strict).
6. `SESSION_SECRET` is any 32 or more random bytes, for example `openssl rand -base64 48`.
7. `SECRETS_KEY` is 32 random bytes, base64, for example `openssl rand -base64 32`; keep a copy in
   your password manager. The Secrets page answers 503 without it.
8. Optional: `ALERTS_TOKEN` for alert issues, `SMTP_URL`, `ALERTS_FROM` and `ALERTS_EMAIL` for
   alert emails, `PLATFORM_APP_CLIENT_ID` and `PLATFORM_APP_PRIVATE_KEY` for releases from the
   Project page, and `DEPLOY_REPORT_SECRET`, the same value as the agent's and the Hub's, for the
   Hub's error rate on the Alerts page.

## What it can do

The console runs privileged with `pid: host` and the Docker socket, so an owner's session is root on
the VPS, and its store holds every project's secrets. Keep the Access policy limited to
organisation members, and `console.env` on the VPS only.
