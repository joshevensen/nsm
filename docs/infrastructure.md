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

- One MySQL instance, one database (schema) per app, one DB user per
  app — not one shared schema. Forge manages database creation on this
  server, same as it would for an app-local database today.
- Rationale for "one instance, separate schemas/users" over either
  extreme (fully shared schema, or fully separate DB servers per app):
  keeps the operational/cost benefits of centralizing (shared backups,
  shared monitoring, shared buffer pool tuning) while keeping a clean
  per-app revoke/kill boundary — see "Kill switch" in `features.md`.
- Redis/Valkey also lives on this server, shared across apps. Since
  queue *workers* are per-app (see Server A), the shared Redis instance
  needs per-app key prefixes / separate Redis DB indexes so queues don't
  collide between apps.
- Firewalled: only Server A (prod app fleet) and Server C (NSM) are
  trusted sources for this server's MySQL/Redis ports, using the same
  "fetch existing firewall rules, append, PUT back" pattern already
  implemented in Fibermade's `database.sh` (`pg_firewall_add_droplet`).

## Server C — NSM + staging

- Hosts the NSM application itself, including **NSM's own database** —
  deliberately kept off Server B so an incident on the shared prod DB
  server doesn't also take down the tool used to diagnose/fix it.
- Hosts staging sites for every app, each staging site with its own
  database (also on this server, not Server B, so staging activity never
  touches production data or production DB load).

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
