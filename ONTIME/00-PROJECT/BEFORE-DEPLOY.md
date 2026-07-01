# Before Deploy — Checklist

Hub: [[OnTime]] · Roadmap: [[ROADMAP]].

**Only genuine deploy-gating infra/config lives here** — things that must be right to ship to Render/Vercel/Neon, regardless of feature work.

> Bugs and tech debt are NOT here — they're normal development, done well before deploy is considered. See [[KNOWN-ISSUES]]. All of those should already be resolved by the time this checklist matters.

> **A demo deploy exists** (Render + Vercel + Neon, see [[STATUS]]) but deliberately does **not**
> satisfy this checklist — it uses a weak `AdminBootstrap` password and free-tier everything.
> Don't treat the demo's existence as this checklist being done.

## Next infra task — in progress 2026-07-01: Google Cloud Run + Supabase

Moving backend off Render → **Google Cloud Run** (`europe-west1`, matched to Supabase's
`aws-0-eu-west-1` region for latency), DB off Neon → back to **Supabase** (reusing the project
from 2026-06-26). Frontend stays on Vercel, unchanged. Deploy is via Cloud Build's
"continuously deploy from a repository" (no local `gcloud` CLI needed) — a push to `OnTime_Backend`'s
`main` branch triggers an automatic build + deploy.

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
