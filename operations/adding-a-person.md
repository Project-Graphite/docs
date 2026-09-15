# Adding a person

## 1. Invite them to the organisation

Org → People → Invite member. Role **Member**, never Owner.

The organisation base permission is `read`, so a new member can see the organisation and nothing
more until you grant it.

## 2. Add them to a team

| Team | Gives | Use for |
| :--- | :--- | :--- |
| `contributors` | read on `actions`, `project-template`, `platform` | everyone |
| `maintainers` | admin everywhere | people who own the platform with you |

Add nearly everyone to `contributors`. `maintainers` is for someone you would trust to change how
every project is built.

## 3. Give them write on their own project

Per repository, not org-wide. Repository → Settings → Collaborators and teams → Add — either the
person directly, or a small team if several people share one project.

They get **write on their own project only**. Read everywhere else.

## 4. Create their registry entry

`platform/people/<handle>.yml` — see [../reference/registry.md](../reference/registry.md).

The handle claims `project-graphite.com/members/<handle>` permanently, so agree it with them first.

```sh
cd platform
python scripts/validate.py
git switch -c feat/add-<handle>
git add people/<handle>.yml
git commit -m "feat: add <name>"
```

On merge their page goes live. It includes projects they maintain and projects that credit their
GitHub username as an author.

## 5. Point them at the documentation

- [../adding-a-project.md](../adding-a-project.md) — the walkthrough
- [../reference/conventions.md](../reference/conventions.md) — naming, PR titles, what goes where
- the contributing guide in `.github`

## Notes

**Members cannot create private repositories.** This is deliberate: on the free plan a private
repository cannot read organisation secrets, so its pipeline fails in a way that looks like a bug.
They can create public ones.

**They cannot push to `platform`.** Contributors have read access. To register a project they open
an issue with the entry filled in and a maintainer commits it. The registry is small and reviewed;
that is the point.

**Rulesets do not protect private repositories on the free plan.** `platform` and
`project-template` have no branch protection available. What protects them is permissions: nobody
outside `maintainers` has write access.
