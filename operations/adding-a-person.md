# Adding a person

## 1. Invite them to the organisation

Org → People → Invite member. Role **Member**, never Owner: an organisation owner is the Hub's
owner too, and can edit, unpublish or delete anything there.

The organisation base permission is `none`: a new member sees the public repositories, and private
ones only through a team.

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

They get **write on their own project repository**, plus the platform repositories through
`contributors`. Read everywhere else.

## 4. Their page appears on the Hub

The Hub syncs the organisation's members from GitHub every hour and on the membership webhook, so
`project-graphite.com/members/<login>` exists within minutes with their GitHub name and avatar,
the projects they own and the projects that credit their username. Nothing is written by hand.

They sign in on the Hub with `?graphite` on any page, then edit their page at
`/members/<login>/edit`: an alias, which gives them `project-graphite.com/<alias>`, headline, bio,
accent, avatar, cover, the blocks in their order, and short links. They can also write posts and
edit the presentation and registry fields of the projects they own.

## 5. Give them console access, if they operate something

Every member reads every console page except Admin, Secrets and Backups. On the console's Admin
page an owner grants, per member, operating a project, that project's secrets, images, the host
shell, or everything. See [console.md](console.md).

## 6. Point them at the documentation

- [../adding-a-project.md](../adding-a-project.md) — the walkthrough
- [../reference/conventions.md](../reference/conventions.md) — naming, PR titles, what goes where
- the contributing guide in `.github`

## Notes

**Members cannot create private repositories.** This is deliberate: on the free plan a private
repository cannot read organisation secrets, so its pipeline fails in a way that looks like a bug.
They can create public ones.

**They cannot push to `main` on the platform repositories.** Contributors push branches and open
pull requests, and a maintainer approves them. They register a project by editing `.graphite.yml`
in their own repository; the registry sync validates it, then opens and merges the `platform` pull
request without maintainer review.

**`main` rulesets.** Each platform repository blocks deletion and force-pushes on `main` and
requires a squash-merged pull request with one approval. Repository admins (organisation owners
and `maintainers`) bypass it, and so does the `graphite-deploy` app on `platform`,
`project-template`, `docs` and `graphite-hub`, so the registry sync and the image tag records keep
pushing.

**Removing a member** from the organisation signs them out of the Hub within a second and takes
their page down on the next sync. Their posts and the versions of what they edited stay in the
Hub's database.
