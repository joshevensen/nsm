# Infrastructure Topology

Three DigitalOcean droplets, managed via Laravel Forge, using DO Reserved
IPs so server identity is stable across a droplet being destroyed and
recreated (e.g. disaster recovery, resizing).

## Server A — Production app fleet

- One Forge site per SaaS app (Novelize, Fibermade, YaFaBa, future apps),
  each isolated as its own Forge site.
- Each app's own PHP version / dependencies managed per-site by Forge, as
  today.
- Per-app queue workers / Horizon daemons run here, one per app codebase
  (Horizon can't be shared across codebases even though Redis is
  shared — see below).
- Per-app scheduler (`schedule:run` every minute), one per site, same
  pattern already used in Fibermade's Forge setup.

## Server B — Database + Redis

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
  queue *workers* are per-app (see Server A), the shared Redis instance
  needs per-app key prefixes / separate Redis DB indexes so queues don't
  collide between apps.
- Firewalled: only Server A (prod app fleet) and Server C (NSM) are
  trusted sources for this server's Postgres/Redis ports, using the same
  "fetch existing firewall rules, append, PUT back" pattern already
  implemented in Fibermade's `database.sh` (`pg_firewall_add_droplet`).

## Server C — NSM + staging

- Hosts the NSM application itself, including **NSM's own database** —
  deliberately kept off Server B so an incident on the shared prod DB
  server doesn't also take down the tool used to diagnose/fix it.
- Hosts staging sites for every app, each staging site with its own
  database (also on this server, not Server B, so staging activity never
  touches production data or production DB load).

## Server sizing

Starting sizes, not permanent commitments — DO droplet resizing is cheap
and low-risk, so the intent is to start smaller than feels safe and size
up on an actual monitoring signal rather than guessing up front.

| Server | Starting size | Cost | Why |
|---|---|---|---|
| A — prod app fleet | 4GB / 2vCPU / 80GB | ~$24/mo | Matches what Novelize's own consolidation plan sized for one app + DB combined; here DB is split off entirely onto Server B, so this server only carries web + PHP-FPM + Horizon + scheduler across all three apps. Per-app DB load is close to idle (`Threads_running` ~1, load average <0.1 per `db-sizing-findings.md`), so this isn't traffic/CPU-bound. The 2nd vCPU exists so one app's deploy (`composer install`/`npm run build`) doesn't stall live requests for the others running concurrently on the same box. |
| B — DB + Redis | 4GB / 2vCPU / 80GB | ~$24/mo | Combined data footprint for a whole app is under 1.5GB (`db-sizing-findings.md`), and the actual failure last time wasn't disk size — `innodb_buffer_pool_size` sat at 128MB despite 3.9GB of unused RAM. The same lesson applies to Postgres's `shared_buffers`/`effective_cache_size`, which ship with similarly conservative defaults — size for cache headroom across three apps' combined working set plus Redis's share, not for the tiny data volume, and set these explicitly rather than leaving Postgres defaults in place. |
| C — NSM + staging | 2GB / 1vCPU / 50GB | ~$12/mo | NSM's traffic is just the owner (and eventually one more team member) — negligible. Staging sites see no real user load either; the only real resource pressure here is periodic build steps during deploys, not sustained traffic. |

**Total: ~$60/mo** for the whole portfolio's infrastructure.

Notes:

- Disk tier matters least of the three dimensions here — even 50GB is
  wildly oversized against <1.5GB/app of actual data. Take whatever disk
  comes bundled with the RAM/CPU tier that fits; don't pay extra for
  disk headroom specifically.
- Watch DO Monitoring for the first few weeks rather than sizing further
  by guesswork. Two things specifically worth alerting on: disk usage
  trend on Server B (the exact failure mode from the original incident),
  and CPU/memory pressure on Server A during overlapping deploys across
  apps.
- If a future app turns out meaningfully heavier than the others, that's
  a Server A resize question, not a reason to give it its own box — the
  point of this topology is that resizing one shared server is simpler
  than managing N separate ones.

## Monitoring

- DO Monitoring (`do-agent`, free) and Forge's own server monitoring are
  the actual monitoring/alerting systems, enabled on all three servers.
- NSM does not reimplement monitoring — it displays values pulled from
  these existing sources (and from Redis/Horizon directly for queue
  stats). See `features.md`.
- An external heartbeat check (something outside all three servers —
  e.g. a free uptime-checking service, or a DO Function if warranted
  later) should ping NSM itself, so there's a way to find out NSM is
  unreachable even when nothing running on NSM's own infrastructure can
  tell you that.

## Explicitly rejected for this topology

See `decisions.md` for full reasoning:

- Kubernetes (DOKS) — operational overhead not justified at this scale.
- DO Functions for application async work (e.g. scheduled emails) — the
  existing scheduler + Horizon on Server A already handles this; Functions
  would add a second deployment/secrets target for no real benefit.
