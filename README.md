# Project Graphite handbook

Project Graphite is an organisation hub for member portfolios and project showcases. Static content
runs on Cloudflare Pages; containerized demonstrations run through Coolify on one OVH VPS.

| Need | Read |
| :--- | :--- |
| Understand the system | [overview.md](overview.md) |
| Add a project | [adding-a-project.md](adding-a-project.md) |
| Understand registry fields | [reference/registry.md](reference/registry.md) |
| Understand hosting | [reference/hosting.md](reference/hosting.md) |
| Prepare the OVH host | [operations/ovh-coolify.md](operations/ovh-coolify.md) |
| Add a member | [operations/adding-a-person.md](operations/adding-a-person.md) |
| Retire a project | [operations/retiring-a-project.md](operations/retiring-a-project.md) |
| Review remaining recommendations | [codex-recs.md](codex-recs.md) |

The private `platform` repository is the source of truth for people, projects, image tags, and
deployment metadata. Production Compose files live with their applications. Secret values never
belong in Git.
