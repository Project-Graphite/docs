# Rotating the deploy app key

The `graphite-deploy` GitHub App is what lets CI write to `platform` and `graphite-hub`. Its private
key is the most sensitive credential in the organisation: it can commit to both.

Rotate it if it leaks, if it appears in a log, or on a schedule you are comfortable with.

## What the app is

| | |
| :--- | :--- |
| App ID | `4923055` |
| Permissions | Contents: read and write. Metadata: read. |
| Installed on | `platform` and `graphite-hub` only |
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

## If a ruleset starts blocking the bot

A ruleset on a repository the app writes to will reject its push with `GH013: Repository rule
violations found`. The app needs to be a bypass actor:

```sh
gh api -X PUT repos/project-graphite/graphite-hub/rulesets/<id> --input ruleset.json
```

with

```json
"bypass_actors": [
  { "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always" },
  { "actor_id": 4923055, "actor_type": "Integration", "bypass_mode": "always" }
]
```

`RepositoryRole: 5` is repository admin — that covers you, not the app. A GitHub App is an
`Integration` and must be listed separately by App ID. This is easy to miss because the failure
looks like a permissions problem rather than a rule problem.

## If the key leaks

1. Delete the key on the app page immediately. Every token minted from it stops working.
2. Generate a new one and update all three locations.
3. Check `platform` and `graphite-hub` commit history for anything the leak period allows.

The blast radius is bounded: Contents write on two repositories, no access to secrets, no admin
rights, no other repository.
