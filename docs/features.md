# NSM Feature Scope

## Server / app lifecycle management

A Forge + DigitalOcean API wrapper, effectively a Terraform-style
provisioning layer purpose-built for this topology. Builds on the
idempotent get-or-create patterns already proven out in Fibermade's
`.github/scripts/` (`lib.sh`, `create-server.sh`, `database.sh`,
`dns.sh`, `spaces.sh`, `stripe-webhook.sh`):

- Spin up a new SaaS: create its Forge site on Server A, its database +
  user on Server B, its staging site + database on Server C, DNS
  records, Spaces bucket, Stripe webhook, initial `.env`.
- Spin down / decommission a SaaS.
- Manage existing apps: redeploy, view deployment status, restart
  queues.

## Environment variable management

- NSM stores env var key/value pairs per app per environment
  (production/staging) in its own database, values encrypted at rest
  (Laravel's `encrypted` cast).
- "Shared" secrets (e.g. one Resend API key or one Stripe test key reused
  across multiple low-traffic apps) are modeled as a single value linked
  into multiple apps' env sets, not copy-pasted per app — rotating it
  once updates everywhere it's linked.
- Changes are pushed to each app's actual Forge-managed `.env` via
  Forge's site-env API endpoint — the same mechanism already implemented
  in Fibermade's `create-server.sh` (`forge_api PUT
  /servers/{id}/sites/{id}/env`), just invoked from NSM instead of a
  one-time provisioning script.
- No value is ever logged or displayed in plaintext in logs — same
  discipline as the existing `mask()` helper in `lib.sh`.
- Decision record: this is a deliberately custom-built module rather
  than adopting a third-party secrets manager (e.g. Infisical). See
  `decisions.md` for the reasoning.

## Monitoring display (read-only)

- Surfaces DO Monitoring and Forge server monitoring data (CPU, memory,
  disk, alerts) per server. NSM does not collect these metrics itself.
- Surfaces Horizon/queue values (queue depth, failed job counts) per app
  by reading the shared Redis instance directly, with each entry
  deep-linking to that app's own real Horizon dashboard for detail. NSM
  does not reimplement Horizon's UI.
- Surfaces per-app scheduled-task status (last run, next run,
  success/failure) for visibility, without owning execution — execution
  stays with each app's own Laravel scheduler on Server A.

## Admin panel for apps and services

- One place to see, per app: which third-party services it uses (Resend,
  Stripe, etc.), current plan/tier, and relevant account-level info.

## Team management

- Purpose: gate *access to which SaaS product's data*, not general RBAC.
- Simple model: a user/SaaS/role pivot. Example case this needs to
  support directly: a user with full access to Novelize and zero access
  to any other app in the portfolio.
- Deliberately not built as a general-purpose teams/roles framework —
  scope is strictly "which app can this person see."

## UI isolation by SaaS

- The UI never shows data from two different SaaS apps on the same
  screen. Every screen operates within a single, explicitly selected
  "current SaaS" context.
- This is enforced naturally by the infrastructure choice of
  separate-database-per-app (see `infrastructure.md`) — a request only
  ever holds one app's DB connection at a time.

## Database GUI (read-only)

- Table browsing with filtering, per app.
- **No SQL console. No write access from the UI.** This is a deliberate
  scope limit — see `decisions.md` for why, given NSM holds credentials
  across the entire portfolio.

## Kill switch

- A fast, one-click way to cut off a single misbehaving app from shared
  infrastructure without affecting the others — e.g. revoke/suspend that
  app's DB user on Server B, or kill its active queries.
- Exists specifically to bound the "noisy neighbor" risk that comes from
  sharing one Postgres instance and one Redis instance across apps: turns a
  potential slow-degradation incident into a quick, contained cutoff.
- Should be built before it's needed, not after.

## Billing / Stripe stats

- Cross-app view of Stripe data (MRR, subscriber counts, etc.) per SaaS.
- Does not require NSM to hold full DB credentials for this specifically
  if it can pull directly from each app's Stripe account/API instead.
