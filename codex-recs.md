# Codex recommendations

Reviewed 2026-09-15 against the local Project Graphite repositories, the public GitHub
organisation and the live site.

## Changes completed in this review

- Member and project pages now have stable canonical routes:
  `/members/<handle>` and `/projects/<slug>`.
- Exact permanent redirects preserve the former `/<handle>` and `/<handle>/<slug>` links.
- A member portfolio includes projects they maintain **or** are credited on, so collaborative work
  is no longer hidden under only one member.
- The second public organisation member has a minimal, clearly editable registry entry.
- The homepage no longer claims that source-only projects have live deployments.
- The build emits `sitemap.xml`, a normal crawler policy, security headers and immutable caching
  for hashed assets.
- GitHub profile enrichment is concurrent, time-bounded and falls back to cached data instead of
  taking the whole deployment down.
- Graphite Hub documentation no longer promises a topology component that is not present.
- Graphite Hub now has an MIT licence.
- The main site repository is now the private `graphite-hub`; its GitHub Actions build,
  Cloudflare Pages deployment and registry synchronization were all verified after the change.

## Private repository governance

The current GitHub plan does not support repository rulesets on private organisation repositories;
the Rules API reports that a plan upgrade or public visibility is required. The repository can stay
private and deploy normally, but "pull requests only" is now a team convention rather than an
enforced control. Keep write access narrow and upgrade to a plan that supports rulesets for private
repositories before relying on required reviews or protected-branch enforcement.

## Decisions still needed

### 1. Confirm every member profile

Registry profiles are an explicit publication and consent boundary. Ask each member to confirm
their preferred handle, name, headline, biography, links and technologies. The initial
`bluocaroot` entry intentionally contains only information already public on GitHub. Do not
automatically publish every organisation member; GitHub membership can be private and membership
alone is not consent to publish a portfolio.

### 2. Define the showcase content contract

Add optional fields for:

- hero screenshot or short demo;
- problem, solution and measurable result;
- each member's roles or contributions;
- repository, live URL and documentation URL;
- long-form Markdown or MDX case study;
- project-specific social image.

The project page should lead with outcome and contribution. Deployment topology belongs in a
secondary architecture section. Until this contract exists, make the public acceptance criteria
match reality: source-only projects do not need a live link, but every listed project should have
a useful README and media when media is meaningful.

### 3. Add deployment targets instead of one universal runtime

Extend `deploy.kind` only when its adapter is implemented and tested:

| Kind | Target | Use |
| :--- | :--- | :--- |
| `static` | Cloudflare Workers Static Assets | static sites and SPAs |
| `worker` | Cloudflare Workers | edge APIs and lightweight full-stack apps |
| `edge-container` | Cloudflare Containers | bursty or stateless existing images |
| `compose` | shared Linux host through Cloudflare Tunnel | long-running and multi-service apps |
| `none` | no runtime | libraries, desktop tools and source-only work |

Cloudflare recommends Workers Static Assets for new sites; Pages remains supported, so migrate the
Graphite Hub when deployment configuration is next touched rather than creating avoidable churn:
https://developers.cloudflare.com/workers/best-practices/workers-best-practices/

Cloudflare Containers are useful but are not a database host. Their filesystem is ephemeral, cold
starts are expected and stateless autoscaling is not yet built in:
https://developers.cloudflare.com/containers/faq/

### 4. Provision the first general-purpose host

Follow `operations/provisioning-the-host.md`, starting with one 4 GB Linux VM and Docker Compose.
Increase to 8 GB only when measured memory use or the number of deployed services requires it.
Use a remotely managed Cloudflare Tunnel, Traefik, the trusted reconciler and separate networks per
project. Keep all inbound firewall ports closed and put administrative endpoints behind Cloudflare
Access.

Before calling the host production-ready:

- enforce CPU, memory and PID limits;
- run project containers without privilege, host mounts or Docker socket access;
- keep the reconciler as the only component allowed to control Docker;
- create a separate database and role per project;
- store secrets outside Git and decrypt them only on the host;
- back up databases daily to off-host object storage;
- run and record a restore test;
- monitor HTTP health, containers, disk, memory, tunnel state and certificate/DNS failures;
- document rollback and host rebuild procedures.

Cloudflare recommends remotely managed tunnels and multiple replicas when availability matters:
https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/

### 5. Harden image promotion

The current short-SHA tags are understandable, but deployment should ultimately record an immutable
OCI digest. Add:

- SBOM generation;
- vulnerability scanning with an explicit severity policy;
- keyless image signing and signature verification on the host;
- digest-based deployment and rollback;
- a GitHub Deployment record and environment URL;
- per-project deployment concurrency;
- post-deploy health checks that automatically restore the prior digest.

Keep the existing least-privilege GitHub App design. Pin third-party Actions to full commit SHAs
and let Dependabot or Renovate propose updates.

### 6. Add preview and status experiences

Static and ordinary Worker projects can use provider preview URLs. Compose projects can later use a
single-level hostname such as `pr-123--atlas.project-graphite.com`; deeper hostnames are not covered
by ordinary wildcard TLS. Expire previews automatically.

Expose deployment metadata—not privileged host data—to the Graphite Hub compiler: deployed digest,
deployment time, health state and last successful check. Never make a green badge from registry
intent alone.

### 7. Finish public presentation

- Add screenshots and a demo for Windhawk Mods.
- Add project and member Open Graph images plus `Person` and `SoftwareApplication` JSON-LD.
- Add filters once the catalogue is large enough to need them.
- Replace the personal Gmail organisation address with a role address on
  `project-graphite.com`.
- Add `SECURITY.md`, issue forms and an optional Discussions area when outside contribution is
  invited.

## Suggested delivery order

1. Confirm member profile consent and add showcase media.
2. Provision and restore-test the Compose host.
3. Promote images by digest and add automatic rollback.
4. Add health/deployment metadata to Graphite Hub.
5. Introduce the `worker` and `static` deployment adapters.
6. Add Cloudflare Containers only for projects whose workload benefits from scale-to-zero.
7. Add a second host, tunnel replica or managed database when a project has a real uptime
   requirement; do not imply high availability while everything still runs on one VM.
