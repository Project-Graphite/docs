# docs

Everything you need to work in Project Graphite, in one place. Private, because it describes the
deployment topology and the operational runbooks.

## Start here

| If you want to | Read |
| :--- | :--- |
| Understand how the pieces fit together | [overview.md](overview.md) |
| **Add a new project, end to end** | [adding-a-project.md](adding-a-project.md) |
| Look up a registry field | [reference/registry.md](reference/registry.md) |
| Know what CI runs and when | [reference/workflows.md](reference/workflows.md) |
| Name something correctly | [reference/conventions.md](reference/conventions.md) |
| Understand hosting and subdomains | [reference/hosting.md](reference/hosting.md) |
| Run the disposable Google Cloud test | [operations/google-cloud-message-lab.md](operations/google-cloud-message-lab.md) |
| Fix something that is broken | [troubleshooting.md](troubleshooting.md) |
| Know why something was built this way | [decisions.md](decisions.md) |
| See the remaining product and hosting recommendations | [codex-recs.md](codex-recs.md) |

Runbooks live in [operations/](operations/).

## Status markers

Documentation is only useful if it distinguishes what exists from what is planned. Every section
that describes something not yet running carries one of these:

> **Built and verified.** This works today and has been exercised end to end.

> **Designed, not provisioned.** The design is settled and the steps are accurate, but nothing is
> running yet. Do not follow these expecting a working system.

If a page carries no marker, it is built and verified.

## The short version

One repository per project. Every part of a project — front end, API and workers — lives in that one
repository, not spread across several repositories.

The platform registry names deployable services. Independent services can use separate Dockerfiles;
services built from the same code can share one image and run different commands.

A project becomes real when it has an entry in the `platform` registry. That one file produces the
card on the site, the project page and, for deployable projects, the hostname and deployment.
