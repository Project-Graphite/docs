# Rotating the deploy app key

The `graphite-deploy` GitHub App lets CI record image tags and merge registry pull requests in
`platform`, read every repository's `.graphite.yml`, and sync the registry to `graphite-hub`. It is
installed on every organisation repository, so its private key is the most sensitive credential in
the organisation: it can commit to any of them.

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

| Location | Why |
| :--- | :--- |
| Org secrets | reach **public** repositories only, on the free plan |
| `platform` repo secrets | `platform` is private, so org secrets cannot reach it |
| `project-template` repo secrets | same reason |

Public project repositories inherit the org secrets automatically and need no setup. Private ones
need their own copy — which is one reason members cannot create private repositories.

## Rotating

**1. Generate a new key.** Org → Settings → Developer settings → GitHub Apps → Graphite Deploy →
Private keys → Generate a private key. A `.pem` downloads. Both keys are valid at once, so nothing
breaks yet.

**2. Update every location.**

```sh
KEY=~/Downloads/graphite-deploy.<date>.private-key.pem

gh secret set PLATFORM_APP_PRIVATE_KEY --org project-graphite --visibility all < "$KEY"
gh secret set PLATFORM_APP_PRIVATE_KEY --repo project-graphite/platform < "$KEY"
gh secret set PLATFORM_APP_PRIVATE_KEY --repo project-graphite/project-template < "$KEY"
```

Read the key **from the file**. Pasting it into a terminal puts it in your shell history and risks
mangling the newlines, which fails with an unhelpful error.

**3. Verify before revoking.** Re-run the latest `platform` Main workflow and confirm its Sync to
Graphite Hub step succeeds; it ends with `Graphite Hub updated.` or
`Registry unchanged; Graphite Hub already current.` Do not skip this — if the new key is wrong you
want both still working.

**4. Delete the old key** on the same GitHub App page.

**5. Delete the local `.pem` files.** They have no further use and are the most likely place for
the key to leak from.

## If the key leaks

1. Delete the key on the app page immediately. Every token minted from it stops working.
2. Generate a new one and update all three locations.
3. Check commit and pull-request history in every organisation repository for anything the leak
   period allows.

The blast radius is Contents and Pull requests write on every organisation repository. The key
gives no access to secrets and no admin rights.
