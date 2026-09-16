# Conventions

## Naming

| Thing | Rule | Example |
| :--- | :--- | :--- |
| Repository | lowercase, hyphenated | `game-forge` |
| Slug | equals the repository name | `game-forge` |
| Service folder | lowercase, describes its role | `frontend`, `server`, `worker` |
| Person handle | lowercase, short, memorable | `amr` |
| Hostname | `<slug>.project-graphite.com` | `game-forge.project-graphite.com` |
| Image | `ghcr.io/project-graphite/<repo>/<service>` | `.../game-forge/server` |

The slug is the repository name, the subdomain and the URL segment. Renaming it later means
renaming the repository, breaking image names and invalidating links. Choose once.

Service folders are named for what they are, not what they use. `server`, not `nestjs`.

## Branches

`type/short-summary`, using the same types as PR titles.

```
feat/project-pages
fix/topology-overflow
chore/bump-vite
```

Branch names are not enforced. The PR title is, because that is what ends up in history.

## Pull request titles

```
type(scope): summary
```

Types: `feat fix chore refactor docs test ci perf revert style build`. The scope is optional.

```
feat: add person and project pages
fix(topology): stop single-service diagrams scaling up
chore: bump vite to 8.0.16
```

Merges are squashed and **the PR title becomes the commit message on `main`**, which is why the
title is checked rather than the branch name. Every repository is configured for squash-only
merges with the title as the commit subject.

## Pull request size

A label is applied automatically from the changed line count, excluding lockfiles.

| Label | Lines |
| :--- | :--- |
| `size/xs` | up to 10 |
| `size/s` | up to 50 |
| `size/m` | up to 200 |
| `size/l` | up to 600 |
| `size/xl` | more |

Large pull requests are not blocked, but they are slower to review and more likely to sit.

## Line endings

Every repository has a `.gitattributes` with `* text=auto eol=lf`. Without it, a Windows checkout
commits CRLF, and a shell script with CRLF fails on Linux with a confusing `bad interpreter` error.
New repositories inherit this from the template.

## Comments in code

The house style is to write none. Name things precisely and extract a well-named function rather
than explaining a block. Put the explanation in the pull request description.

The exception is anything load-bearing for a tool rather than a human: `# noqa`, `// eslint-disable`,
`# type: ignore`, licence headers, codegen markers.

## Secrets

Never in a repository. Not in code, config, commit messages, logs or a pull request description.

The registry names the environment variables a service needs; runtime values live in namespace
Kubernetes Secrets and build credentials live in GitHub secrets. Every push is scanned over full
history, and a committed secret fails the build.

If you commit one by accident, rotate it first and rewrite history second. Rewriting alone does not
help: it is already in the reflog, in forks and in anything that cloned it.

## Reviews

Your own project repository is yours. Merge what you like there.

`platform`, `actions` and `graphite-hub` need review from `@project-graphite/maintainers`, because a
change there affects everyone else. CODEOWNERS routes this automatically.

## What belongs where

| Change | Repository |
| :--- | :--- |
| Project code | that project |
| How every project is built or checked | `actions` |
| What a project is, or how it is deployed | `platform` |
| How the site looks | `graphite-hub` |
| How the organisation works | `docs` |

If you are editing a workflow inside a project repository, stop. It almost certainly belongs in
`actions`.
