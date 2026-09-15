# Decisions

Why things are the way they are. Mostly so that a later change is made deliberately rather than by
someone who assumed it was an accident.

---

## One repository per project, not per service

A project with a front end, an API and a worker keeps all three in one repository, in separate
top-level folders.

Splitting a project across repositories means a change to a shared contract lands in two places at
two times, and neither repository tells the whole story. One repository per project keeps the
history of a thing in one place, which is what makes it readable as a portfolio piece later.

The cost is that CI has to work out which parts changed. That is the detect step, and it is about
thirty lines.

## A folder with a Dockerfile is a service

Nothing declares the service list. CI derives it from the filesystem.

The alternative — a config file listing services — drifts. Someone adds a folder and forgets the
list, and the failure is a service that silently never deploys. Deriving it means the two cannot
disagree, and adding a service is adding a folder.

The registry does name services, but only to say how they are exposed, and validation checks those
names against the filesystem.

## Subdomains for applications, paths for pages

`atlas.project-graphite.com` runs the application. `project-graphite.com/projects/atlas` is the page
about it.

Serving an application under a path means it has to be **built** knowing where it is mounted —
Vite `base`, React Router `basename`, and every absolute asset path. That couples every project to
the platform, and it fails in the worst way: the page loads, the styles do not.

It also puts every project on one origin, sharing cookies, `localStorage` and service worker scope.
With several people contributing, one project could read another's stored data. Subdomains are
separate origins and the browser enforces it.

Paths as pages give the URL structure without any of that, because the page is data Graphite Hub
already has.

## Subdomains stay one level deep

Cloudflare Universal SSL on the free plan covers the apex and first-level subdomains only.
`atlas.amr.project-graphite.com` would have no valid certificate; deeper levels need Advanced
Certificate Manager, which is paid.

If two people ever want the same slug, namespace the slug rather than the domain.

## `actions` is public

It has to be. **A public repository cannot call a reusable workflow in a private one** — GitHub
blocks it so outside collaborators cannot indirectly read private workflow logs.

Project repositories are public, so `actions` must be. It contains workflow definitions and no
secrets, so there is nothing to protect.

This was found the hard way: `graphite-hub` failed at startup with zero jobs and no error text, and
took four probe workflows to isolate.

## Project repositories are public, and members cannot create private ones

Three reasons, in order of how much they bite:

1. **Organisation secrets do not reach private repositories on the free plan.** A private project
   repository gets an empty `PLATFORM_APP_ID` and a deploy that fails confusingly.
2. **Actions minutes.** Private repositories share one 2,000 minute pool; public are unlimited.
3. **The code is the track record.** A portfolio of private repositories is a list of screenshots.

`platform` and `project-template` stay private and carry their own repository secrets. That is the
deliberate exception, and it costs two commands each.

## Image tags live outside the registry entry

`projects/<slug>.yml` is human-authored. `tags/<slug>.json` is bot-written.

Keeping them in one file would mean every deploy rewrites a file people are editing, producing
conflicts and unreadable diffs. Split, a registry diff is a human decision and a tags diff is a
deployment, and `git log tags/` reads as deployment history.

## Deployment is pull-based

The host polls `platform` every sixty seconds rather than CI pushing to the host.

No SSH keys in CI, no inbound ports on the box, and contributors never touch it. Rollback is
`git revert` in `platform`. It is the same model a GitOps controller uses, in about fifteen lines
of shell.

## Production compose is generated, not written

Contributors write `compose.yaml` for local development. The deployed compose file is rendered from
the registry.

If contributors wrote production compose, reviewing it would mean checking every pull request for a
Docker socket mount or a host bind, forever. Generating it means every service gets the same
hardening — read-only root, dropped capabilities, non-root user, memory and CPU limits — and nobody
can opt out by accident.

It also means a contributor writes about fifteen lines of YAML instead of learning the deployment.

## Resource sizes are fixed presets

`small` and `medium`, not free-form numbers. The host is shared, and one project should not be able
to take it down. A preset is also a decision someone can make without knowing what the box has.

## Graphite Hub is prerendered

Every route is written to a real HTML file at build time.

This is not an optimisation. It is what makes the site work on a static host: `/amr/` resolves to a
real file, so deep links work without SPA-fallback rewrites, and unknown paths genuinely miss and
return a real 404. Getting per-route titles and crawlable content is the bonus.

The alternative — SPA fallback for everything — returns 200 for every URL that has ever been typed,
which is worse than a 404 in every way that matters.

## React rather than Astro

Graphite Hub was built in Astro first and converted.

Astro shipped zero JavaScript, which is genuinely better for a content site. But this is the
repository that gets edited most often, and being fluent in it matters more than 70 KB. The
conversion cost an hour at the start; it would have cost a great deal more later.

Prerendering recovered the important crawlability and deep-link properties. The hydrated React
application still ships JavaScript, which is an accepted trade-off rather than a zero-JavaScript
site.

## Tailwind with a single token set

The palette is declared once in an `@theme` block, which generates both the CSS custom properties
and the utilities. So `bg-paper`, `text-muted`, `stroke-line` and `fill-surface` all read the same
tokens, so every component reads the same palette.

Light and dark swap by redefining those properties. There is not a single `dark:` prefix in the
codebase.

## Deployment topology must be generated

> **Designed, not implemented.**

When deployment topology returns to project pages, it must be generated from the registry rather
than drawn by hand. A hand-drawn architecture diagram drifts from reality; the registry already
contains the services, routes, ports and database needed to render an accurate secondary
architecture section. The current site deliberately does not claim that this component exists.

## Nothing is deleted, only marked

An abandoned project becomes `status: archived` and keeps its card. The live link disappears,
because only `live` produces one.

A visible progression over time is worth more than a curated handful, and the alternative failure —
a card advertising a URL that stopped resolving — is worse than an honest label.

## No host until a project needs one

The container platform is designed and unbuilt on purpose.

Every static project is free on Cloudflare Pages. Provisioning a box before anything needs it means
maintaining a box for no reason. The design is settled so that building it is an afternoon, not a
rethink.
