# Retiring a project

Nothing gets deleted. A project that is no longer maintained is marked, not removed — a visible
progression over time is worth more than a curated handful, and a dead link discovered later is
worse than an honest label.

## 1. Change the status

`platform/projects/<slug>.yml`:

```yaml
status: archived
```

The card keeps its place with a grey dot reading "archived", and **the live link disappears** —
only `live` produces one. That alone prevents the most common rot, which is a card advertising a
URL that stopped resolving months ago.

## 2. Stop the deployment

```yaml
deploy:
  kind: none
```

Remove the `host`, `services` and `database` keys; `kind: none` accepts nothing else.

On merge, the compose project is removed from the host on the next reconcile and its resources come
back. For a `kind: pages` project, delete the Pages project in Cloudflare separately — the registry
does not manage it.

## 3. Clean up what the registry no longer covers

- delete the DNS record if one was created by hand
- remove the vanity Bulk Redirect entry
- remove the project secrets file from the host
- archive the repository on GitHub: Settings → Archive this repository

Archiving makes it read-only and greys it in listings, while keeping the code and history readable.

## 4. Leave the database alone until you are sure

Dropping a database is not reversible. Take a dump first, keep it somewhere off the host, and drop
it later if you still want to.

```sh
ssh graphite
docker compose -p _data exec -T db pg_dump -U postgres <slug> | gzip > ~/<slug>-final.sql.gz
```

## If the project should disappear entirely

Set `listed: false` rather than deleting the entry. The registry keeps the record, the site stops
showing it, and the hostname stays claimed so nobody reuses it by accident.

Deleting the file is also fine, but then nothing stops a future project taking the same slug and
inheriting stale links.
