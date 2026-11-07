# Rotating the deploy app key

The `graphite-deploy` GitHub App lets CI write `IMAGE_TAG` into `platform` and tag releases, the
registry sync read every repository's `.graphite.yml` and merge its pull request, the Hub read the
organisation and commit registry edits back into `.graphite.yml`, and the console release a tag
from its Project page. It is installed on every organisation repository, so its private key is the
most sensitive credential in the organisation: it can commit to any of them.

Rotate it if it leaks, if it appears in a log, or on a schedule you are comfortable with.

## What the app is

| | |
| :--- | :--- |
| App ID | `4923055` |
| Permissions | Contents: read and write. Pull requests: read and write. Metadata: read. |
| Installed on | all repositories (the registry sync reads every `.graphite.yml`) |
| Webhook | disabled |

The App ID is not secret — it is public installation metadata. The private key is.

## Where the credentials live

| Copy | Used for |
| :--- | :--- |
| organisation secrets `PLATFORM_APP_CLIENT_ID` and `PLATFORM_APP_PRIVATE_KEY` | every workflow: the deploy record, the changelog, **PR** and **Sync registry** in `platform` |
| `/graphite-hub/PLATFORM_APP_CLIENT_ID` and `PLATFORM_APP_PRIVATE_KEY` in the console's store | the Hub's reads of the registry, members and releases, and write-back |
| `PLATFORM_APP_*` in `/etc/graphite/console.env` | releases and `ENABLED` from the console's Project page; optional |

Every public repository inherits the organisation secrets. On the free plan they do not reach
private repositories, which is one reason members cannot create private ones.

## Rotating

**1. Generate a new key.** Org → Settings → Developer settings → GitHub Apps → Graphite Deploy →
Private keys → Generate a private key. A `.pem` downloads. Both keys are valid at once, so nothing
breaks yet.

**2. Update every copy.**

```sh
KEY=~/Downloads/graphite-deploy.<date>.private-key.pem

gh secret set PLATFORM_APP_PRIVATE_KEY --org project-graphite --visibility all < "$KEY"
```

Read the key **from the file**. Pasting it into a terminal puts it in your shell history and risks
mangling the newlines, which fails with an unhelpful error. Then paste the file's contents into
`/graphite-hub/PLATFORM_APP_PRIVATE_KEY` on the console's Secrets page and redeploy the Hub, and
replace it in `console.env` if it is set there.

**3. Verify before revoking.** Run `platform` → Actions → **Sync registry** by hand and confirm it
ends with `Registry already current.` or a merged pull request, and check the Hub's `/status` still
shows a sync after the redeploy. Do not skip this — if the new key is wrong you want both still
working.

**4. Delete the old key** on the same GitHub App page.

**5. Delete the local `.pem` files.** They have no further use and are the most likely place for
the key to leak from.

## If the key leaks

1. Delete the key on the app page immediately. Every token minted from it stops working.
2. Generate a new one and update every copy.
3. Check commit and pull-request history in every organisation repository for anything the leak
   period allows.

The blast radius is Contents and Pull requests write on every organisation repository, including
`platform`, where a commit to `deploy/<slug>/values` deploys. It reaches no secret value: those are
in the console's store, not in any repository. The key gives no admin rights.
