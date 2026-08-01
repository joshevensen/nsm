# Decisions

Record of the notable choices made while scoping NSM, and the reasoning
behind them — so the "why" isn't lost later.

## Standardize on Postgres, migrate off MySQL

All apps run on a single Postgres instance (DB Server), not MySQL, and
not a mix of both. Before this decision, the portfolio was already
split — Fibermade provisions DO Managed Postgres for production
(`fibermade/.github/scripts/create-server.sh`), while Novelize runs
MySQL 5.7 in production. A shared DB server needs one engine, not two:
running both permanently on DB Server would mean twice the patching,
tuning, and monitoring surface for no benefit, working directly against
NSM's simplicity-first priority.

**Chose Postgres over MySQL**, for one concrete, forward-looking reason:
`pgvector` gives every app a path to embeddings/RAG-style AI features
using infrastructure already being run, with no separate vector store to
stand up later. MySQL's equivalent (HeatWave vector search) is an Oracle
Cloud-only managed feature, not available in the self-hosted community
MySQL that Forge/DO would provision here — so this isn't available on
the MySQL path at all, self-hosted.

**Chose to migrate Novelize rather than migrate Fibermade to MySQL**,
since Novelize is the smaller, older, lower-activity codebase (Laravel
8, <1GB of data, idle load per `db-sizing-findings.md`) — meaningfully
cheaper to move than Fibermade, which is the actively developed,
larger, newer (Laravel 13) app. YaFaBa and any future app should be
built on Postgres from the start; no migration question there.

**What the Novelize migration actually requires** (found via a scoped,
non-exhaustive code search — treat as a floor, not a guarantee there's
nothing else):

- At least one known MySQL-specific query needs rewriting:
  `->orderBy(DB::raw('RAND()'))` in `NameController.php` →
  Postgres's `RANDOM()`.
- A full regression pass (`php artisan test`) against Postgres before
  cutover, not just a syntax-level check.
- This lands at the same time as an open, separate question — whether
  Novelize's very old dependency set (Laravel 8, `doctrine/dbal` 2.x,
  `fideloper/proxy`, `fruitcake/laravel-cors`, `intervention/image`
  2.5.1 — all abandoned or pre-dating PHP 8.3/8.4-era changes) actually
  runs cleanly on the PHP 8.4 target. Since Novelize needs to be touched
  and fully retested for the Postgres move regardless, verifying PHP 8.4
  compatibility in the same pass is more efficient than two separate
  verification cycles later.
- Same dump/restore discipline as Novelize's own prior MySQL 5.7 → 8.0
  migration plan (`db-app-consolidation-migration-plan.md`) applies here
  too: test the restore against a throwaway Postgres 18 instance before
  the real cutover window, not live.

This also makes the earlier "MySQL 8.0 vs 8.4" sizing question moot —
there's no MySQL instance being provisioned at all.

## Shared DB server accepted, with mitigations for its known risk

One Postgres instance on DB Server serves every app (separate
database/user per app, not a fully shared database). This concentrates
risk that prior per-project infra planning had argued against at the
single-app level (see Novelize's own
`docs/devops/db-app-consolidation-migration-plan.md`, which rejected
even a same-droplet DB for one app in favor of full isolation). NSM
supersedes that reasoning deliberately, not by oversight: at
sub-1,000-users-per-app scale, the operational cost of maintaining 3-5
separate DB servers outweighs the isolation benefit, and the specific
prior incident that motivated caution (a disk quietly filling up) is
directly addressed by DO Monitoring + a generous disk buffer, which
didn't exist last time.

What this doesn't fully solve: **noisy-neighbor contention** (one app's
lock contention or traffic spike affecting others sharing the instance)
is a different failure mode than disk space, and monitoring alone only
detects it after the fact rather than preventing it. The kill switch
(see `features.md`) is the accepted mitigation — fast containment
instead of prevention.

## Single production app server, not one per app

Further consolidation beyond the DB question: all production app code
also runs on one droplet (App Server), meaning a single droplet failure
takes the entire portfolio offline simultaneously, not just one app.
Accepted because recovery is bounded — the GitHub-Action-provisions-a-
replacement-droplet pattern already validated in Novelize's
consolidation plan applies here too, so worst case is "minutes to
redeploy," not "unknown."

## Dedicated Marketing Server, not colocated on App Server

Every app's marketing + blog site uses Nuxt with Nuxt Studio
(https://nuxt.studio/) for visual content editing. Studio's live editor
requires an actual SSR route running — a pure static export (no live
Node process at all) would mean giving up the visual editor entirely, so
"just static-generate it and skip running Nuxt altogether" was
considered and rejected as infeasible for what's actually wanted here.

Given a live Nuxt process is required per app regardless, the remaining
question was where it runs. Rejected colocating it on App Server, in
favor of a dedicated Marketing Server, for two reasons that both pointed
the same direction:

- **Isolation**: colocating would mean a whole-droplet failure on App
  Server takes down every app's marketing/blog site along with the app
  itself — the same kind of cascading failure that prompted this
  question in the first place (an SSL cert issue had previously taken
  down Novelize's marketing site without affecting the app; colocating
  would remove that boundary at the droplet level even though cert
  failures specifically stay isolated regardless, since certs are
  per-domain not per-codebase).
- **Cost**: bumping App Server a full tier to absorb the added Node
  processes (4GB→8GB, +$24/mo) costs more than giving the lightweight
  Nuxt processes their own small server (+$12/mo), since most of what
  App Server's extra tier buys is PHP-FPM/Horizon headroom the Nuxt
  processes don't need.

This is the standard shape for **every** app's marketing site, not a
Novelize-specific exception — confirmed this is a portfolio-wide need,
not a one-off. Real visitor traffic hits pre-rendered static output;
the live SSR route only serves the owner's own Studio editing sessions,
which is why this server can start small. Static generation happens in
CI (GitHub Actions), not on the droplet, so the server only ever runs
the lightweight runtime, never the build step.

**Started at 2GB/1vCPU (~$12/mo) deliberately without strong confidence
in that number** — there's no benchmark data for Nuxt Content v3 +
Studio's actual per-site idle footprint, and 1 vCPU means no true
concurrency across apps' processes. Explicitly a "start here, validate
against real DO Monitoring numbers as apps are added, upgrade before it
becomes a problem" choice, same policy as every other server's sizing —
just with a wider-than-usual error bar called out up front.

## Custom-built env var manager, not a third-party secrets tool

Evaluated adopting Infisical (mature, self-hostable, MIT-licensed) as
NSM's secrets layer instead of building one. Decided to build a minimal
version into NSM directly: NSM is already going to be a Forge API
wrapper for provisioning, and Fibermade's `create-server.sh` already has
working code pushing rendered `.env` content to Forge's site-env
endpoint — storing key/value pairs in NSM's own encrypted-at-rest DB and
pushing them via that same endpoint is a small increment on
infrastructure being built anyway, not a new subsystem. This trade only
makes sense because of NSM's threat model (no enterprise compliance
requirement, no external team needing scoped access) — would not
recommend rebuilding this for a higher-stakes context.

## No Kubernetes (DOKS)

Rejected for this scale. DOKS earns its cost when you need autoscaling,
many replicas, or complex service meshes — none of which apply when
every app is resource-light and under ~1,000 users. The cost side is
real: containerizing every app, operating cluster upgrades, ingress,
cert-manager, persistent volumes — ongoing tax for a solo developer, and
it would discard the working Forge-based tooling already built and
debugged. Revisit only if a specific app's compute becomes an actual
bottleneck, which is not the current or anticipated situation.

## No DO Functions for application async work

Considered using DO Functions for per-app scheduled/async work (e.g.
Novelize's progress-report emails). Rejected: this work is already well
served by the existing Laravel scheduler + Horizon queue on App Server,
which is infrastructure already being paid for and already getting
visibility built into NSM. Moving it to Functions would introduce a
second deployment target with its own env var sync path and its own
success/failure visibility to build — real added complexity for no
volume-driven benefit at this scale.

**Where Functions could still make sense**: an external heartbeat check
of NSM's own availability, run somewhere outside all four managed
servers, since nothing hosted on NSM's own infrastructure can tell you
when NSM itself is down. Not committed to yet — a free external
uptime-check service may cover this without needing Functions at all.

## No SQL console / no write access in the database GUI

NSM necessarily holds credentials with real access to every app's
production data across the whole portfolio. A full ad-hoc SQL console
would make a single compromised NSM login a breach of everything, not
just one app — disproportionate to the "niche SaaS, not enterprise
data" threat model this is meant to match. Scoped down to read-only
table browsing with filtering. If a one-off write/fix is ever needed,
it should go through a deliberate, explicit action rather than a bare
SQL box — not designed yet, not needed yet.

## NSM open source, not sold as a SaaS product

Considered turning NSM into a product sold to other solo developers
running their own niche SaaS portfolios. Decided against it: selling it
means becoming custodian of *other people's* production DB credentials
and API keys, which is a meaningfully higher security/support/liability
bar than "internal tool for my own three apps," and not a responsibility
wanted here. Building NSM to be sellable today would also violate its
own "simplicity first, no speculative future-proofing" principle —
multi-tenant isolation between unrelated customers, provider-
agnosticism, billing, and support are all real added scope with no
current justification.

Chose to open source the code instead, once it exists — this doesn't
meaningfully hurt security as long as no secret ever lives in the repo
(everything stays in env vars or NSM's own encrypted DB columns, the
same discipline already used in Fibermade's provisioning scripts).
Security-through-obscurity was never real protection to begin with. The
honest cost is that auth/validation logic becomes readable in advance,
typically offset by more eyes finding bugs if the project gets any
attention.

If a future version of "sell this" is ever revisited, license choice
matters at that point (plain MIT vs. something source-available-but-
not-freely-hostable) — not a decision needed now.

## NSM's own security is hardened above the portfolio's baseline

Because NSM concentrates access to every app in the portfolio, its own
login is hardened beyond what any individual app needs: 2FA required
(Fortify, already in the stack), and — since reserved IPs are already in
use for server identity — consider firewalling NSM's own web UI to known
IPs rather than leaving it open to the public internet. This is not
"enterprise security theater" — it's proportionate to NSM being a single
point of access to everything else, which none of the individual apps
are.
