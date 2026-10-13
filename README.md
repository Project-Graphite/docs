# Project Graphite handbook

Project Graphite is an organisation hub for member portfolios and project showcases. Static sites
run on Cloudflare Pages; containerized projects run on one OVH VPS. `platform` deploys both.

| Need | Read |
| :--- | :--- |
| Understand the system | [overview.md](overview.md) |
| Understand why it works this way | [decisions.md](decisions.md) |
| Add a project | [adding-a-project.md](adding-a-project.md) |
| Understand registry fields | [reference/registry.md](reference/registry.md) |
| Understand hosting | [reference/hosting.md](reference/hosting.md) |
| Follow naming, pull request and review rules | [reference/conventions.md](reference/conventions.md) |
| Understand the shared workflows | [reference/workflows.md](reference/workflows.md) |
| Run the VPS | [operations/vps.md](operations/vps.md) |
| Use the console | [operations/console.md](operations/console.md) |
| Manage secrets | [operations/managing-secrets.md](operations/managing-secrets.md) |
| Operate Graphite Tracker | [operations/graphite-tracker.md](operations/graphite-tracker.md) |
| Add a member | [operations/adding-a-person.md](operations/adding-a-person.md) |
| Rotate the deploy app key | [operations/rotating-the-deploy-app.md](operations/rotating-the-deploy-app.md) |
| Retire a project | [operations/retiring-a-project.md](operations/retiring-a-project.md) |
| Fix a known failure | [troubleshooting.md](troubleshooting.md) |
| Move to k3s later | [plans/k3s-transition.md](plans/k3s-transition.md) |
| Review remaining recommendations | [recommendations.md](recommendations.md) |

The `platform` repository holds people, the project registry, each project's production values,
image tags, the core chart and the VPS edge. Secret values live only in the owners-only `secrets`
repository.

The handbook is also readable in the console's **Docs** tab.
