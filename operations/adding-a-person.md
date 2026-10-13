# Adding a person

## 1. Invite them to the organisation

Org → People → Invite member. Role **Member**, never Owner.

The organisation base permission is `none`: a new member sees the public repositories, and private
ones only through a team. Private repositories, such as `secrets`, are for owners only.

## 2. Add them to a team

| Team | Gives | Use for |
| :--- | :--- | :--- |
| `contributors` | write on `actions`, `platform`, `project-template`, `docs`, `.github`, `graphite-hub`; `main` takes only reviewed pull requests | everyone |
| `maintainers` | admin on `actions`, `platform`, `project-template`, `.github`, `graphite-hub`, `docs` | people who own the platform with you |

Add nearly everyone to `contributors`. `maintainers` is for someone you would trust to change how
every project is built.

## 3. Give them write on their own project

Per repository, not org-wide. Repository → Settings → Collaborators and teams → Add — either the
person directly, or a small team if several people share one project.

They get **write on their own project only**. Read everywhere else.

## 4. Their registry entry is created for them

The next registry sync creates `platform/people/<lowercase-login>.yml` with `profile: github` and
fills `name`, `headline` and `links` from their GitHub profile. It refreshes them on every later
sync. The sync runs on any `.graphite.yml` push in the organisation, or by hand from
`platform` → Actions → **Sync registry**.

To write their profile by hand instead, they set `profile: manual` and edit the file on a
branch of `platform`, changing only `people/<handle>.yml`, then open a pull request for a
maintainer to approve.
`bio` and `stack` are always written by hand. See [../reference/registry.md](../reference/registry.md).

Their page includes projects they maintain and projects that credit their GitHub username as an
author.

## 5. Point them at the documentation

- [../adding-a-project.md](../adding-a-project.md) — the walkthrough
- [../reference/conventions.md](../reference/conventions.md) — naming, PR titles, what goes where
- the contributing guide in `.github`

## Notes

**Members cannot create private repositories.** This is deliberate: on the free plan a private
repository cannot read organisation secrets, so its pipeline fails in a way that looks like a bug.
They can create public ones.

**They cannot push to `main` on the platform repositories.** Contributors push branches and open
pull requests, and a maintainer approves them. They register a
project by editing `.graphite.yml` in their own repository; the registry sync validates it, then opens and merges the `platform` pull request without maintainer
review. Hand-written person entries go through a reviewed pull request.

**`main` rulesets.** Each platform repository blocks deletion and force-pushes on `main` and
requires a squash-merged pull request with one approval. Organisation admins and the
`graphite-deploy` app bypass it, so the registry sync, tag records and changelog keep pushing.
