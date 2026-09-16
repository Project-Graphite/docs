# Decisions

## One repository per project

A frontend, API, and worker that form one product stay in one repository. This keeps their shared
contracts and portfolio history together. Deployable service names remain explicit in the platform
registry, and CI publishes a matching immutable image for each one.

## Applications use subdomains; showcases use paths

`atlas.project-graphite.com` is the application. `project-graphite.com/projects/atlas` describes it.
Subdomains isolate browser cookies, storage, and service workers. Hostnames stay one level below the
apex so Cloudflare's standard certificate coverage is sufficient.

## Public project code, private control plane

Project repositories and reusable actions are public. This supports the portfolio, avoids private
Actions-minute limits, and allows public repositories to call shared workflows. `platform` remains
private because it contains deployment topology; it still contains no secrets.

## Human registry data and deployment tags are separate

People edit `projects/<slug>.yml`; automation edits `tags/<slug>.json`. A deploy cannot conflict with
a person editing project metadata, and tag history remains a readable rollout record.

## Kubernetes deployment is registry-generated

Contributors maintain Docker Compose only for local development. The platform registry generates
production Deployments, Services, StatefulSets, Jobs, policies, quotas, and routing. This keeps one
reviewable security and resource model across every project and prevents application repositories
from introducing privileged production settings.

## One namespace per project

Namespace isolation is the smallest model that allows multiple projects on one low-cost cluster
without mixing configuration, Secrets, service discovery, quotas, or ordinary network traffic. It
is soft multi-tenancy for trusted members, not a boundary for hostile code.

## No public Kubernetes ingress service

Cloudflare Tunnel reaches an internal gateway Service. Projects use ClusterIP Services only. This
avoids a billable cloud load balancer and leaves no application port open to the internet.

## Fixed resource presets

`tiny`, `small`, and `medium` are easier to review than arbitrary numbers and prevent accidental
resource claims from bypassing the starter cluster's constraints. Namespace quotas provide a
second bound. Capacity changes remain explicit cost decisions.

## Deployment application remains manual initially

CI publishes immutable images, records tags, validates the registry, and renders Kubernetes YAML.
It does not hold cluster credentials or apply changes. Manual server-side dry run, review, and apply
remain in place until rollback, restoration, secret rotation, and cost behavior are proven. A
pull-based reconciler can automate this later with narrowly scoped permissions.

## Graphite Hub is prerendered

Every route becomes a real HTML file. Deep links and per-page metadata work on a static host, and
unknown paths return a genuine 404. React is retained for contributor familiarity; prerendering
recovers the important static-site properties.

## Archived projects remain visible

An abandoned project becomes `archived` and loses its live deployment link, but its authorship and
case study remain. This preserves the organisation's history without advertising a dead service.
