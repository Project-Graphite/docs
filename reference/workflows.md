# Shared workflows

The public `actions` repository holds reusable workflows. Projects call them at `@main`; a
repository that must hold back pins a commit SHA at the call site.

`reusable-pipeline.yml` chains detect, checks, images, deploy, changelog, gitleaks and PR
conventions behind boolean switches (`all`, `checks`, `deploy`, `changelog`, `secrets-scan`,
`conventions`; all `true` except `all`). The template's `pipeline.yml` calls it. Using it is
optional: a repository can call the individual workflows instead.

```text
pull request     -> checks -> image build without push
main push        -> checks -> sha-<short> and latest images -> reusable-deploy -> IMAGE_TAG in platform deploy/<slug>/values
platform push    -> Deploy -> render compose.yaml over shared/ -> signed POST to the deploy agent
                                -> docker compose up --wait -> route -> DNS -> rollout reported to the Hub
                             -> or build the site -> wrangler direct upload to Cloudflare Pages
main push        -> Changelog -> next version tag -> GitHub Release -> the Hub records it from the webhook
.graphite.yml push -> the Hub's webhook -> sync-registry dispatch -> platform Sync registry -> projects/<slug>.yml
```

`reusable-deploy.yml` writes the tag only when the registry entry is `compose` or `pages` and
`deploy/<slug>/` exists in `platform`. `reusable-changelog.yml` runs the Hub's release sync for
the calling repository: it tags the next version from the conventional commits since the last tag
and publishes a GitHub Release with generated notes for each new tag; the Hub's changelog reads
the release from the organisation webhook.

A service's `.build-contexts` file adds named build contexts, one `name=path` per line relative to
the repository root. The build passes them to Docker, and change detection rebuilds the service
when any of those paths change. PixelPocket's frontend uses `emulator=./gb-emulator`.

Organisation defaults remain read-only. Only the image build receives `packages: write`. The
`graphite-deploy` GitHub App is installed on every repository with contents and pull-request
write, because the registry sync reads every `.graphite.yml`, CI writes `IMAGE_TAG` into
`platform`, and the Hub commits registry edits back into `.graphite.yml`. Deploy credentials live
in `platform`'s `production` environment, which only `main` can use.
