# Infrastructure Topology

Four DigitalOcean droplets, managed via Laravel Forge, using DO Reserved
IPs so server identity is stable across a droplet being destroyed and
recreated (e.g. disaster recovery, resizing).

## App Server — production app fleet

- One Forge site per SaaS app (Novelize, Fibermade, YaFaBa, future apps),
  each isolated as its own Forge site.
- Each app's own PHP version / dependencies managed per-site by Forge, as
  today.
- Per-app queue workers / Horizon daemons run here, one per app codebase
  (Horizon can't be shared across codebases even though Redis is
  shared — see below).
- Per-app scheduler (`schedule:run` every minute), one per site, same
  pattern already used in Fibermade's Forge setup.
- Does **not** host marketing/content sites — see Marketing Server.

## DB Server — database + Redis

- **One Postgres instance (Postgres 18), one database per app, one DB
  user per app** — not one shared database. Forge manages database
  creation on this server, same as it would for an app-local database
  today. Postgres, not MySQL, standardized across every app — see
  `decisions.md` for why (pgvector availability was the deciding
  factor).
- `pgvector` extension enabled on this instance, so any app can adopt
  embeddings/RAG-style AI features later without a separate vector store.
- Rationale for "one instance, separate databases/users" over either
  extreme (fully shared database, or fully separate DB servers per app):
  keeps the operational/cost benefits of centralizing (shared backups,
  shared monitoring, shared memory/cache tuning) while keeping a clean
  per-app revoke/kill boundary — see "Kill switch" in `features.md`.
- Redis/Valkey also lives on this server, shared across apps. Since
  queue *workers* are per-app (see App Server), the shared Redis instance
  needs per-app key prefixes / separate Redis DB indexes so queues don't
  collide between apps.
- Firewalled: only App Server and NSM Server are trusted sources for
  this server's Postgres/Redis ports, using the same "fetch existing
  firewall rules, append, PUT back" pattern already implemented in
  Fibermade's `database.sh` (`pg_firewall_add_droplet`).

## Marketing Server — marketing + blog sites (Nuxt / Nuxt Studio)

- Hosts every app's marketing + blog site: Nuxt with hybrid rendering —
  content pages pre-rendered to static output, plus one live SSR route
  for Nuxt Studio's auth/preview so content can be edited visually
  (https://nuxt.studio/). Real visitor traffic hits the static output;
  the live SSR route only serves the owner's own editing sessions.
- Static generation (`nuxt generate`/build) happens in CI (GitHub
  Actions), not on this droplet — only the built static output plus the
  lightweight Nitro runtime are deployed here. Keeps this server's load
  to "run a small idle process per app," not "compile a Nuxt site per
  app on every deploy."
- One process per app (Novelize, Fibermade, YaFaBa, future apps), all on
  this one shared droplet — mirrors how App Server shares one droplet
  across every app's Laravel site.
- Domain pattern per app: root domain (e.g. `getnovelize.com`) → this
  server's Nuxt site; `app.` subdomain (e.g. `app.getnovelize.com`) →
  App Server's Laravel site. Kept as two separate servers deliberately,
  not colocated on App Server, so a whole-droplet failure on one doesn't
  take down the other — see `decisions.md`.
- This is the standard shape for **every** app's marketing site, not a
  Novelize-specific exception — every app is expected to want the same
  Nuxt Studio authoring experience.
- Does not apply to Fibermade's creator storefronts — those are dynamic,
  per-creator app data and stay served by App Server's Laravel site, not
  this server. (Open item: still resolving the exact routing/naming
  split between Fibermade's marketing site and creator storefronts.)

## NSM Server — NSM + staging

- Hosts the NSM application itself, including **NSM's own database** —
  deliberately kept off DB Server so an incident on the shared prod DB
  server doesn't also take down the tool used to diagnose/fix it.
- Hosts staging sites for every app, each staging site with its own
  database (also on this server, not DB Server, so staging activity
  never touches production data or production DB load).
- Also hosts each app's staging counterpart of its Marketing Server
  Nuxt/Studio process, for previewing content changes before they go
  live — same prod/staging split as everything else on this server.
- Named for NSM specifically, not "Staging Server," deliberately: NSM is
  the highest-value target in the portfolio (it can touch every app's
  data), so the name is meant to cue that this box needs the extra care
  even though staging sites live here too — see `decisions.md`.

## Object storage

- **One DO Spaces bucket per app** (and one per staging counterpart),
  not one shared bucket — same isolation principle as DB Server's
  per-app database + user, extended to file storage.
- Each bucket gets its own scoped access key, minted via the DO API
  (`POST /spaces/keys` with a `grants: [{bucket, permission}]` body)
  rather than a `fullaccess` key — the same mechanism already
  implemented in Fibermade's `spaces_key_ensure`/`spaces_bucket_ensure`
  (`spaces.sh`). NSM's provisioning extends this pattern to every app
  rather than inventing a new one.
- Why not one shared bucket: a shared bucket would force either a
  `fullaccess` key per app (one app's compromised credentials expose
  every app's files) or accept losing the ability to revoke one app's
  storage access without touching the others — the same kill-switch
  reasoning applied to DB Server's per-app database users. Separate
  buckets also prevent a wrong-prefix bug in one app's code from
  reading/overwriting another app's files, and let per-app bucket
  settings (public vs. private, CORS) differ without one app's needs
  constraining another's.
- Not a cost-driven decision — Spaces' flat per-bucket fee makes
  multiple buckets cheap in absolute terms regardless.

## Server sizing

Starting sizes, not permanent commitments — DO droplet resizing is cheap
and low-risk, so the intent is to start smaller than feels safe and size
up on an actual monitoring signal rather than guessing up front.

| Server | Starting size | Cost | Why |
|---|---|---|---|
| App Server | 4GB / 2vCPU / 80GB | ~$24/mo | Matches what Novelize's own consolidation plan sized for one app + DB combined; here DB (and now marketing/content) are split off onto their own servers, so this server only carries web + PHP-FPM + Horizon + scheduler across all apps. Per-app DB load is close to idle (`Threads_running` ~1, load average <0.1 per `db-sizing-findings.md`), so this isn't traffic/CPU-bound. The 2nd vCPU exists so one app's deploy (`composer install`/`npm run build`) doesn't stall live requests for the others running concurrently on the same box. |
| DB Server | 4GB / 2vCPU / 80GB | ~$24/mo | Combined data footprint for a whole app is under 1.5GB (`db-sizing-findings.md`), and the actual failure last time wasn't disk size — `innodb_buffer_pool_size` sat at 128MB despite 3.9GB of unused RAM. The same lesson applies to Postgres's `shared_buffers`/`effective_cache_size`, which ship with similarly conservative defaults — size for cache headroom across apps' combined working set plus Redis's share, not for the tiny data volume, and set these explicitly rather than leaving Postgres defaults in place. |
| Marketing Server | 2GB / 1vCPU / 50GB | ~$12/mo | Real visitor traffic hits pre-rendered static output, not the live Nitro process — the SSR route only serves the owner's own Studio editing sessions, so this stays cheap to run. Rough estimate, not a benchmark: no hard numbers yet for Nuxt Content v3 + Studio's actual idle footprint per site, and only 1 vCPU means no true concurrency across apps' processes. Explicitly a "start here and watch it" size — validate against real DO Monitoring numbers as apps are added, and upgrade before it becomes a problem rather than after. |
| NSM Server | 2GB / 1vCPU / 50GB | ~$12/mo | NSM's traffic is just the owner (and eventually one more team member) — negligible. Staging sites and staging content previews see no real user load either; the only real resource pressure here is periodic build steps during deploys, not sustained traffic. |

**Total: ~$72/mo** for the whole portfolio's infrastructure.

Notes:

- Disk tier matters least of these dimensions — even 50GB is wildly
  oversized against <1.5GB/app of actual data. Take whatever disk comes
  bundled with the RAM/CPU tier that fits; don't pay extra for disk
  headroom specifically.
- Watch DO Monitoring for the first few weeks rather than sizing further
  by guesswork. Specifically worth alerting on: disk usage trend on DB
  Server (the exact failure mode from the original incident), CPU/memory
  pressure on App Server during overlapping deploys across apps, and
  memory pressure on Marketing Server as more apps' Nuxt processes are
  added to it (the least-validated number in this table).
- If a future app turns out meaningfully heavier than the others, that's
  a resize question for the relevant server, not a reason to give it its
  own box — the point of this topology is that resizing one shared
  server is simpler than managing N separate ones.

## Monitoring

- DO Monitoring (`do-agent`, free) and Forge's own server monitoring are
  the actual monitoring/alerting systems, enabled on all four servers.
- NSM does not reimplement monitoring — it displays values pulled from
  these existing sources (and from Redis/Horizon directly for queue
  stats). See `features.md`.
- An external heartbeat check (something outside all four servers — e.g.
  a free uptime-checking service, or a DO Function if warranted later)
  should ping NSM itself, so there's a way to find out NSM is
  unreachable even when nothing running on NSM's own infrastructure can
  tell you that.

## Explicitly rejected for this topology

See `decisions.md` for full reasoning:

- Kubernetes (DOKS) — operational overhead not justified at this scale.
- DO Functions for application async work (e.g. scheduled emails) — the
  existing scheduler + Horizon on App Server already handles this;
  Functions would add a second deployment/secrets target for no real
  benefit.
