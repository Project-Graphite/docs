# Retiring a project

Retirement has a reversible registry phase and a destructive runtime phase. Do not combine them.

## Archive it

In the project's `.graphite.yml`:

```yaml
status: archived
deploy:
  kind: none
```

Push it to `main`; the registry sync rewrites `platform/projects/<slug>.yml`. Keep the repository,
authors, showcase and tag history.

## Stop it

Set `enabled: false` in `platform/deploy/<slug>/values.yaml` and merge. **Deploy** removes its route
and stops its containers. Its volumes and `secrets.env` stay on the VPS.

## Preserve state

Before deleting anything, export the database, record the last image tag, and decide how long to
keep the export. An export that has not been restored in a test is not verified.

## Remove it

Each step deletes data or changes DNS; prepare the exact targets and approve each one:

1. On the VPS: `sudo docker volume ls --filter label=com.docker.compose.project=<slug>`, then
   `sudo docker volume rm` each volume, and `sudo rm -r /srv/graphite/<slug>`.
2. In `secrets`: delete `<slug>.env`.
3. In `platform`: delete `deploy/<slug>/`.
4. In Cloudflare: delete the hostname's DNS record.

Do not reuse a retired hostname immediately. The archived registry entry keeps the Hub history but no
longer claims the hostname.
