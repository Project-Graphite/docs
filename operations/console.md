# The console

`https://console.project-graphite.com` is the one place to look at and operate the VPS. Code:
[`console`](https://github.com/project-graphite/console). Production:
`platform/deploy/console/`.

| Page | Owners | Contributors |
| :--- | :--- | :--- |
| Overview | host CPU, memory, swap and disk; host usage charts; storage by images, volumes, container layers and build cache; one row per project with status, CPU, memory and storage, filterable and pinnable | their own projects |
| Project | usage chart with a line per container; containers; images and volumes with sizes; wake | the same, without wake |
| Container | usage chart, details and networks, live logs, start, stop, restart, wake, a shell, its ports | usage chart, details, live logs |
| Images | every image, its size and users; remove dangling images | — |
| Volumes | every volume, its project, size and whether it is in use | — |
| Shell | a root shell on the VPS | — |
| Docs | the handbook, rendered from the `docs` repository | the same |

Usage charts cover the last hour, 5 hours, day, 7 days or 30 days. Hover a chart for the time and
every line's value; the line nearest the pointer is highlighted. Pins and the chosen range are kept
in your browser.

## Ports

A container's web port opens at `/open/<container>/<port>/`, for example MinIO's console. Apps that
build absolute links from `/` may not render there; use an SSH tunnel for those. For any TCP port,
the container page shows the forwarding command:

```sh
ssh -L 5432:<container ip>:5432 ubuntu@135.125.254.59
```

## Sign-in

The console's host is proxied through Cloudflare, and Cloudflare Access admits only organisation members. The console then signs in with GitHub: an
organisation admin is an owner, and anyone else sees the projects they own or author in the
registry. Sessions last 1 hour. Owner actions (start, stop, restart, wake, prune, terminals, the
host shell, open ports) recheck the role with GitHub before they run, so a demoted owner loses them
within a minute and their open terminals close at their next refused action or hourly recheck.

## Setting it up

Once, before the console's first deploy. The values in the last column go in `secrets/console.env`.

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

## What it can do

The console runs privileged with `pid: host` and the Docker socket, so an owner's session is root on
the VPS. Keep the Access policy limited to organisation members and the GitHub OAuth App's secret in
the `secrets` repository only.
