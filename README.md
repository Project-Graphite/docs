# Project Graphite handbook

Project Graphite is an organisation hub for member portfolios and project showcases. Static content
runs on Cloudflare Pages; containerized applications share one namespace-isolated GKE cluster.

| Need | Read |
| :--- | :--- |
| Understand the system | [overview.md](overview.md) |
| Add a project | [adding-a-project.md](adding-a-project.md) |
| Understand registry fields | [reference/registry.md](reference/registry.md) |
| Understand hosting | [reference/hosting.md](reference/hosting.md) |
| Create the GKE lab | [operations/google-cloud-gke.md](operations/google-cloud-gke.md) |
| Add a member | [operations/adding-a-person.md](operations/adding-a-person.md) |
| Retire a project | [operations/retiring-a-project.md](operations/retiring-a-project.md) |
| Review remaining recommendations | [codex-recs.md](codex-recs.md) |

The private `platform` repository is the source of truth for people, projects, image tags, runtime
configuration names, and generated Kubernetes resources. Secret values never belong in Git.
