# The console

`https://console.project-graphite.com` is the one place to look at and operate the VPS. Code:
[`console`](https://github.com/project-graphite/console). Values:
`platform/deploy/console/values.yaml`.

| Page | Owners | Contributors |
| :--- | :--- | :--- |
| Overview | host CPU, memory, swap, disk and 24h CPU; every project's containers with state, CPU, memory and image | their own projects' containers |
| Container | usage history, details and networks, live logs, start, stop, restart, wake, a shell, its ports | usage history, details, live logs |
| Images | every image, its size and users; remove dangling images | — |
| Volumes | every volume, its project, size and whether it is in use | — |
| Shell | a root shell on the VPS | — |

## Ports

A container's web port opens at `/open/<container>/<port>/`, for example MinIO's console. Apps that
build absolute links from `/` may not render there; use an SSH tunnel for those. For any TCP port,
the container page shows the forwarding command:

```sh
ssh -L 5432:<container ip>:5432 deploy@ssh.project-graphite.com
```

## Sign-in

Cloudflare Access admits only organisation members. The console then signs in with GitHub: an
organisation admin is an owner, and anyone else sees the projects they own or author in the
registry. Sessions last 8 hours.

## What it can do

The console runs privileged with `pid: host` and the Docker socket, so an owner's session is root on
the VPS. Keep the Access policy limited to organisation members and the GitHub OAuth App's secret in
the `secrets` repository only.
