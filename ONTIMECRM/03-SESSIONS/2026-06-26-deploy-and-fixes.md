# First free deploy (Render + Vercel + Neon) + small UX fixes — 2026-06-26

Session log (snapshot, not updated later). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** get a free, working demo deploy live for a job interview the next morning, then knock
out a few small UX papercuts found while testing the live deploy.

**Done:**
- **Backend deployed to Render** (free tier, Docker, Frankfurt region). Health check path fixed
  from the autofilled `/healthz` (doesn't exist) to the real `/health` (`Program.cs`). 8 env vars
  set: `ASPNETCORE_ENVIRONMENT`, `ConnectionStrings__DefaultConnection`, `Jwt__Key/Issuer/Audience`,
  `AdminBootstrap__Email/Password`, `Cors__AllowedOrigins__0`.
- **Database: Supabase → Neon migration.** Originally provisioned Supabase (Ireland region,
  "Transaction pooler" / Supavisor, port 6543). Every deploy failed with
  `Npgsql.NpgsqlException: Timeout during reading attempt` on the very first schema-check query —
  confirmed via a local `docker run postgres:16-alpine psql ...` test that the same query took
  **59ms** directly from the developer's machine, proving Supabase itself was healthy and the
  problem was specifically the Render↔Supabase network path (cross-region, Frankfurt↔Ireland, and/or
  Supavisor's prepared-statement incompatibility with Npgsql — tried `Server Compatibility
  Mode=NoTypeLoading`, `Max Auto Prepare=0`, and a longer timeout first, all failed identically).
  Switched to **Neon** (Frankfurt region — same region as the Render service), which worked on the
  first deploy attempt with no special connection-string flags needed.
- **Frontend deployed to Vercel.** `VITE_API_BASE_URL` env var wired to the Render URL.
- **Vercel SPA 404 fixed**: `OnTimeCRM_Frontend/vercel.json` didn't exist, so direct navigation to
  any client-side route (e.g. typing `/dashboard` in the address bar) 404'd. Added a catch-all
  rewrite to `index.html`.
- **VehicleProposalTable single-color auto-fill**: when a vehicle version has exactly one
  available external/internal color, the color `Select` now auto-fills and disables — same pattern
  already used for single-brand auto-lock. New Proposal modal widened 720px → 1040px (Valor/Desconto
  columns were visually cramped).
- **New "Como Funciona" page** (`HelpPage.tsx`, route `/help`): a static AntD `Steps` walkthrough
  of the 5 main flows (Pipeline de Clientes, Propostas, Vendas, Objetivos, Amigos), for onboarding
  and demo narration. New i18n keys in both locales + the pt-PT fallback.

**Findings / gotchas:**
- A "Host can't be null" Npgsql error on first deploy attempt was simply an empty/misformatted
  `ConnectionStrings__DefaultConnection` env var — always re-check the literal saved value in
  Render's UI, not just what you typed, after navigating away from the form.
- Render's autofilled Health Check Path (`/healthz`) doesn't match this API's actual endpoint
  (`/health`, from `app.MapHealthChecks("/health")` in `Program.cs`) — always verify against source
  rather than trusting platform defaults.
- Supabase's dashboard messaging about IPv4/IPv6 ("dedicated IPv4 add-on") was a red herring for
  this failure — the shared pooler already accepts IPv4 free of charge; the actual failure mode was
  a slow/stalled read on an already-established connection, not a connection-refused/DNS issue.
- Testing DB connectivity from the developer's own machine (raw TCP via `Test-NetConnection`, then
  a full query via a disposable `docker run postgres:16-alpine psql ...` container) is a fast way to
  prove "the database is fine, the problem is in the path between hosting platforms" without
  needing to instrument the deployed app.

**Didn't work / dead ends:**
- `Server Compatibility Mode=NoTypeLoading` + `Max Auto Prepare=0` (targeting a suspected
  prepared-statement incompatibility with Supabase's Supavisor pooler) — same timeout, same line.
- `Timeout=60;Command Timeout=60` (giving the first cross-region query more time) — same timeout
  again, just confirmed it wasn't merely "30s is too tight," it was actually stalling.
- Considered Supabase's "Session pooler" (port 5432) as an alternative to the Transaction pooler,
  but this Supabase project's dashboard only exposed the Transaction-pooler connection details —
  no visible mode switch.

**Next:**
- Update `Cors__AllowedOrigins__0` on Render with the final Vercel production URL (was a placeholder
  during the early steps of this session — confirm it's the real one before the interview).
- `AdminBootstrap__Password` is intentionally weak (`teste`) for the demo — see the new warning in
  [[BEFORE-DEPLOY]]; do not treat this deploy as meeting that checklist.
- Warm up the Render free-tier service a few minutes before the interview (cold start after 15 min
  idle is ~30-50s).
- [[ROADMAP]] Phase 4 — new item: configurable lead-temperature rules + manual override (today
  `DealTemperature` is 100% automatic, hardcoded thresholds, no way for a salesperson to override).

**Links:** [[ROADMAP]] Phase 4 · [[BEFORE-DEPLOY]] · [[STATUS]] · [[ARCHITECTURE]]
