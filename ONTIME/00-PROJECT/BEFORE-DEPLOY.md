# Before Deploy — Checklist

Hub: [[OnTime]] · Roadmap: [[ROADMAP]].

**Only genuine deploy-gating infra/config lives here** — things that must be right to ship, regardless of feature work.

> Bugs and tech debt are NOT here — they're normal development, done well before deploy is considered. See [[KNOWN-ISSUES]]. All of those should already be resolved by the time this checklist matters.

> **Live demo deploy** ✅ 2026-07-01 — **Vercel (frontend) + Supabase (DB) + Google Cloud Run
> (backend API)**, replacing the earlier Render + Vercel + Neon setup (see history below). Still
> deliberately does **not** satisfy this whole checklist as a "ready for real customers" gate — see
> the remaining unchecked items below (CORS is now locked to the real Vercel origin, but the
> `AdminBootstrap` password, JWT secrets etc. are demo-strength, not audited production secrets).
> Don't treat the demo's existence as this checklist being done.

## Current stack (since 2026-07-01): Vercel + Supabase + Google Cloud Run

- **Frontend**: Vercel, auto-deploys on every push to `OnTime_Frontend`'s `main` branch.
  URL: `https://getontime.vercel.app` (a free `*.vercel.app` subdomain — no custom domain owned yet).
- **Backend**: Google Cloud Run (`europe-west1`/Belgium — closest Google region to Supabase's
  `aws-0-eu-west-1`/Ireland). Deploys via a Cloud Build trigger ("continuously deploy from a
  repository") on every push to `OnTime_Backend`'s `main` — no local `gcloud` CLI needed for
  day-to-day deploys. URL: `https://ontime-901852227634.europe-west1.run.app`.
- **Database**: Supabase (reusing the project created 2026-06-26). Connect via the
  **Session Pooler** (port `5432`), not the Transaction Pooler (`6543`) — see below for why.

**Why this over the previous Render + Neon setup:** Render's free tier had a slow/unreliable
network path to Supabase (the original reason for the Neon detour) and cold-starts after 15 min
idle; Cloud Run's free tier (2M requests/month, region-matched to Supabase) worked reliably once
the pooler-mode bug below was found and fixed, and Supabase gives `pg_cron` out of the box — needed
for [[ROADMAP]]'s last remaining Sprint 1 item (the scheduled-jobs trigger), which Neon doesn't
offer as easily. Not clearly "better" in the abstract — better *for this project's specific needs*
(pg_cron) and once the two bugs below were actually fixed, which Render+Supabase never got past.

**Two real bugs hit and fixed while wiring this up:**
1. **Npgsql + Supabase Transaction Pooler timeout.** The Transaction Pooler (port 6543) defaults
   to IPv6, and Cloud Run's connection to it hung indefinitely reading the query response (not a
   connection failure — auth succeeded, only the data stream stalled) until Npgsql's 30s
   `CommandTimeout` fired. Fixed by switching to the **Session Pooler** (same host, port `5432`),
   which works over IPv4. If you ever see `Npgsql.NpgsqlException: Exception while reading from
   stream` / `TimeoutException: Timeout during reading attempt` against Supabase from a new host,
   check this first before assuming it's a app-level or generic network issue.
2. **`DatabaseInitializer` couldn't run against Supabase at all** — see next section, this was a
   real portability bug, not Supabase-specific.

## Open idea — real EF Core Migrations (not started, needs /grill-me)

Discussed 2026-07-01, deliberately deferred rather than rushed: replace today's drift-detect +
Production-refuses-to-start guard (below) with **real EF Core Migrations**
(`dotnet ef migrations add`), which would auto-generate the actual `ALTER TABLE`/`CREATE TABLE`
correction script instead of requiring a hand-written one. This directly **contradicts
CLAUDE.md's non-negotiable rule #3** ("No EF migrations") — that rule was a deliberate early
decision (see [[../04-DECISIONS/2026-05-30-data-layer]]) and reversing it needs to be a conscious
call, not a drive-by change.

**Why this is riskier than most features:** the app now holds real customer data in Supabase
(since the first live deploy, same date). The cutover requires manually inserting a row into
Postgres's `__EFMigrationsHistory` table on **both** local docker-compose Postgres and the
production Supabase database, to tell EF "this schema already exists, don't try to CREATE TABLE
things that are already there." That's hand-written SQL against production — exactly the kind of
action that needs a safety net first.

**Supabase backup situation (checked 2026-07-01):** the free tier has **no automatic backups or
point-in-time recovery** — Pro tier ($25/mo) adds daily backups (7-day retention); PITR is a
further paid add-on. On the free tier, the only safety net before touching production schema is a
**manual `pg_dump`** taken right before the cutover.

**Before implementing:** run `/grill-me` on this (this sprint or next — not yet scheduled) to lock
down: exact cutover sequence, who takes the `pg_dump` and when, whether to upgrade to Supabase Pro
first given real customer data now exists, and whether "real Migrations" fully replaces or
coexists with the current Production-refuses-to-start guard during a transition period.

## Open idea — a deploy skill/agent that generates the schema-correction script automatically

Also raised 2026-07-01, not started: today, when schema drift is detected in Production, the app
just refuses to start and logs what's wrong — a human still has to read the drift, write the
`ALTER TABLE`/`CREATE TABLE` SQL by hand, and run it against Supabase before redeploying (see the
guard below). The idea: a Claude Code skill or agent, run as part of the deploy step, that reads
the EF model diff and **generates that correction script automatically** — turning today's manual
step into a reviewed-but-automated one. Would likely become unnecessary if the EF Core Migrations
idea above ships first (migrations already generate their own correction script) — worth deciding
which path to take before building both.

## Infra / data — schema init on managed Postgres hosts ✅ fixed 2026-07-01

`DatabaseInitializer` used to call `EnsureDeletedAsync()`/`EnsureCreatedAsync()` (i.e. `DROP
DATABASE` + recreate) whenever schema drift was detected. This **fails outright on managed hosts
like Supabase/Neon/RDS**: they give you one fixed, shared database (Supabase's is always literally
named `postgres`) — Postgres refuses to `DROP DATABASE` the one you're currently connected to, and
even a second connection wouldn't have permission on a managed host. Worse, EF Core's own
`EnsureCreatedAsync` "does this DB already have tables" heuristic scans *all* schemas, not just
`public` — so it silently concluded our schema was already set up, because Supabase always
provisions its own `auth`/`storage`/`realtime` schemas with tables, even in a brand-new project.

**Fixed:** `DatabaseInitializer` now manages the `public` schema directly (`IRelationalDatabaseCreator.CreateTablesAsync`
+ a scoped `DROP SCHEMA public CASCADE; CREATE SCHEMA public;` instead of ever touching the whole
database). Verified locally: dropped a column, restarted the container, confirmed it recreated
just that column without needing to drop/recreate the database.

**Production data-safety guard (added the same day, before the first real customer used this):**
once real data exists in Supabase, silently wiping `public` on drift would destroy it. So in
Production (`isProduction: true` passed from `Program.cs`'s `app.Environment.IsProduction()`),
if schema drift is detected **and at least one of our tables already exists** (meaning there could
be real data), `InitializeAsync` throws instead of touching anything — the app refuses to start
rather than risk data loss. The very first deploy against a brand-new, completely empty Supabase
project is still safe and unaffected (zero of our tables exist yet, nothing to lose) — the guard
only blocks *later* deploys that introduce a schema change against a database that's already in
use. Adding a column/table in Production from now on requires a deliberate manual step (a
hand-written `ALTER TABLE`/`CREATE TABLE` run directly against Supabase) until the project adopts
real EF Core migrations — see [[ARCHITECTURE]] for why there are none today.

## Config / security
- [ ] CORS falls back to `AllowAnyOrigin()` when `Cors:AllowedOrigins` is empty — set real origins.
- [ ] `Jwt:Key/Issuer/Audience` must be production secrets, not dev defaults.
- [ ] `AdminBootstrap` seeds a demo admin from config only if no companies exist — set real secrets, don't ship the demo account.

## Features that block billing
- [ ] Payment integration is a stub (`InitiateAsync` throws `PAYMENT_PENDING`) — required before charging anyone. See [[PAYMENTS]].

## Gate
- [ ] All [[KNOWN-ISSUES]] resolved.
- [ ] Full test suite green (`dotnet test`). See [[TESTING]].

## Related
[[KNOWN-ISSUES]] · [[ROADMAP]] · [[ARCHITECTURE]] · [[PAYMENTS]] · [[OnTime]]
