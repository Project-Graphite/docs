# Retiring a project

Retirement has a reversible registry phase and a destructive runtime phase. Do not combine them.

## Archive it

In the project's `.graphite.yml`:

```yaml
status: archived
deploy:
  kind: none
```

Push it to `main`; the Hub's webhook dispatches the registry sync, which rewrites
`platform/projects/<slug>.yml`, and the Hub stops probing the host. Keep the repository, authors
and showcase; the project stays on the Hub as archived, with its releases.

## Stop it

Set `ENABLED=false` in `platform/deploy/<slug>/values` and merge. **Deploy** removes its route
and stops its containers. Its volumes, its backups and `/etc/graphite/<slug>.env` stay. Deleting
`deploy/<slug>/` has the same effect while the registry still says `compose`; once the entry says
`none`, delete the folder in a later pull request.

A `pages` site has nothing on the VPS. `ENABLED=false` only stops new uploads; the site keeps
serving until its Pages project is deleted.

## Preserve state

Before deleting anything, press **Back up now** for the project on the console's Backups page,
confirm the archive verified, download it, and decide how long to keep it. An archive that has not
been imported in a test is not verified beyond its checksums.

## Remove it

Each step deletes data or changes DNS; prepare the exact targets and approve each one:

1. On the VPS: `sudo docker volume ls --filter label=com.docker.compose.project=<slug>`, then
   `sudo docker volume rm` each volume, `sudo rm -r /srv/graphite/<slug>`, and
   `sudo rm -r /var/lib/graphite/backups/<slug>` once the download is safe.
2. In the console: select every secret under `/<slug>/` on the Secrets page and delete them, which
   removes `/etc/graphite/<slug>.env`. The compose file must be gone from the VPS first, or the
   delete is refused for names it still requires.
3. In `platform`: delete `deploy/<slug>/`.
4. In Cloudflare: delete the hostname's DNS record.

For a `pages` site, skip steps 1 and 2: delete `deploy/<slug>/` in `platform`, then in Cloudflare →
Workers & Pages delete the Pages project, and delete the hostname's CNAME.

Do not reuse a retired hostname immediately. The archived registry entry keeps the Hub history but no
longer claims the hostname.
