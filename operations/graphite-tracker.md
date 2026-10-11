# Graphite Tracker operations

Production runs at `https://graphite-tracker.project-graphite.com` from
`platform/deploy/graphite-tracker/values.yaml`. Every start applies pending Prisma migrations, so
review migrations before merging to `main`.

## Values

| Name | Value | Required |
| :--- | :--- | :--- |
| `POSTGRES_PASSWORD` | generated secret | yes |
| `AUTH_ACCESS_TOKEN_SECRET` | separate long generated secret; also signs digest unsubscribe links | yes |
| `EMAIL_HOST` | `smtp.gmail.com` | yes |
| `DEFAULT_FROM_EMAIL` | `Graphite Tracker <projectgraphite.org@gmail.com>` | yes |
| `EMAIL_PORT` | `587`; `465` uses TLS from the start | no, `587` |
| `EMAIL_HOST_USER` | `projectgraphite.org@gmail.com` | for Gmail |
| `EMAIL_HOST_PASSWORD` | Gmail app password without spaces | for Gmail |
| `TMDB_READ_ACCESS_TOKEN` | TMDB read access token | no |
| `GAME_SOURCE` | `igdb` or `rawg` | no, `igdb` |
| `IGDB_CLIENT_ID`, `IGDB_CLIENT_SECRET` | IGDB credentials | when `GAME_SOURCE` is `igdb` |
| `RAWG_API_KEY` | RAWG key | when `GAME_SOURCE` is `rawg` |
| `DISABLED_SOURCES` | comma-separated source IDs | no |

They live in `secrets/graphite-tracker.env`; see [managing-secrets.md](managing-secrets.md).
Rotating `AUTH_ACCESS_TOKEN_SECRET` signs everyone out and breaks every unsubscribe link already
sent.

## Gmail sender

`projectgraphite.org@gmail.com` is the shared sender; each project uses its own display name.
Gmail SMTP needs an app password:

1. Sign in as `projectgraphite.org@gmail.com` and turn on 2-Step Verification.
2. Open [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords), create one
   named after the project, and store its 16 characters without spaces as `EMAIL_HOST_PASSWORD`.

The account sends about 500 messages in any rolling 24 hours across every project. Move to a
dedicated provider before digests approach that.

## Roles

| Role | Can |
| :--- | :--- |
| Member | everything a reader does |
| Administrator | moderate reports and reviews, deactivate members, see private profiles and hidden entries, see failed release emails on `/admin` |
| System manager | everything an administrator can, plus appoint administrators and turn adult content off site-wide |

The system manager can only be appointed from the VPS:

```sh
sudo docker exec -it graphite-tracker-app-1 npm run system-manager:grant --workspace server -- MANAGER_EMAIL
sudo docker exec -it graphite-tracker-app-1 npm run system-manager:grant --workspace server -- NEW_EMAIL --transfer
```

The account must be registered and verified. `--transfer` hands the role over and also recovers it
when the old account is lost. Administrators are appointed in `/admin` → **Users**; adult content
is switched off for everyone in `/admin` → **Site**.

Only when email cannot be delivered, verify an account directly:

```sh
sudo docker exec -it graphite-tracker-postgres-1 psql -U graphite -d graphite_tracker -c "UPDATE users SET verified_at = now() WHERE email = 'MANAGER_EMAIL' AND verified_at IS NULL RETURNING handle;"
```

## Rolling back

1. Pick the last good `sha-…` tag.
2. Check no migration blocks it: rolling back the image does not roll back the database. Images
   built before `20260927100000_user_roles` cannot run against a database that has it.
3. Set `services.app.tag` in `platform/deploy/graphite-tracker/values.yaml` and merge.
4. Remove the pin once the fix ships.

## Backups

There is no database backup yet. Add one, like Strata's `backup` service, and test a restore before
inviting real users.
