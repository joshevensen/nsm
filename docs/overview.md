# NSM (Niche SaaS Manager) — Overview

## What this is

NSM is a personal infrastructure control plane for managing a small
portfolio of niche SaaS products (currently Novelize, Fibermade, and
YaFaBa, with more expected over time) built and operated by a solo
developer / small team, on top of DigitalOcean + Laravel Forge.

The premise: niche SaaS products built by solo developers/small teams are
becoming more common (accelerated by AI-assisted development), and each
one individually is small — sub-1,000 users, light resource needs, low
traffic. Managing infrastructure, secrets, and access per-app doesn't
scale well for one person as the number of apps grows. NSM centralizes
that operational overhead into one tool instead of duplicating it per app.

## Guiding priorities (in order)

1. **Simplicity first.** Prefer the simplest reliable option over the
   theoretically-best one. No premature abstraction, no building for
   hypothetical future requirements (including a hypothetical future
   where NSM itself becomes a multi-tenant product — see
   `decisions.md`).
2. **Cost**, after simplicity.
3. **Security, at a level proportionate to the actual data.** This is
   explicitly *not* enterprise-sensitive data — niche SaaS products with
   small user bases, not handling regulated or high-value data. Security
   effort should match that reality, not over-engineer for a threat
   model that doesn't apply. The one deliberate exception: NSM itself
   becomes the highest-value target in the whole portfolio (it can touch
   every app's data), so NSM's own access controls are hardened
   specifically because of that concentration, not because of the
   underlying apps' sensitivity.

## What NSM is not

- Not a Kubernetes-based platform — evaluated and rejected for this
  scale (see `decisions.md`).
- Not built around serverless/DO Functions for application async work —
  evaluated and rejected (see `decisions.md`).
- Not designed to be sold as a multi-tenant SaaS to other developers —
  a deliberate choice to avoid taking on custody of other people's
  production credentials (see `decisions.md`). The code is intended to
  be open-sourced instead.
- Not a full DB admin tool — no SQL console, no write access via the UI.
  Read-only table browsing only (see `features.md`).
- Not a replacement for DO Monitoring, Forge server monitoring, or
  Horizon's own dashboard — NSM aggregates and displays data from these,
  it does not reimplement them.
