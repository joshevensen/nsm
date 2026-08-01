# Decisions

Record of the notable choices made while scoping NSM, and the reasoning
behind them — so the "why" isn't lost later.

## Shared DB server accepted, with mitigations for its known risk

One MySQL instance on Server B serves every app (separate schema/user
per app, not a fully shared schema). This concentrates risk that
previously prior per-project infra planning had argued against at the
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
also runs on one droplet (Server A), meaning a single droplet failure
takes the entire portfolio offline simultaneously, not just one app.
Accepted because recovery is bounded — the GitHub-Action-provisions-a-
replacement-droplet pattern already validated in Novelize's
consolidation plan applies here too, so worst case is "minutes to
redeploy," not "unknown."

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
served by the existing Laravel scheduler + Horizon queue on Server A,
which is infrastructure already being paid for and already getting
visibility built into NSM. Moving it to Functions would introduce a
second deployment target with its own env var sync path and its own
success/failure visibility to build — real added complexity for no
volume-driven benefit at this scale.

**Where Functions could still make sense**: an external heartbeat check
of NSM's own availability, run somewhere outside all three managed
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
