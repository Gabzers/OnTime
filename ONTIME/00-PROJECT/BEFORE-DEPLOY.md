# Before Deploy — Checklist

Hub: [[OnTime]] · Roadmap: [[ROADMAP]].

**Only genuine deploy-gating infra/config lives here** — things that must be right to ship to Render/Vercel/Neon, regardless of feature work.

> Bugs and tech debt are NOT here — they're normal development, done well before deploy is considered. See [[KNOWN-ISSUES]]. All of those should already be resolved by the time this checklist matters.

> **A demo deploy exists** (Render + Vercel + Neon, see [[STATUS]]) but deliberately does **not**
> satisfy this checklist — it uses a weak `AdminBootstrap` password and free-tier everything.
> Don't treat the demo's existence as this checklist being done.

## Next infra task (before the next deploy round)
- [ ] **Move backend hosting + DB to Google Cloud + Supabase, keep the current frontend
  (Vercel)** (noted 2026-06-27, not started). I.e.: backend off Render → Google Cloud (likely
  Cloud Run, given no background workers exist today — see [[NOTIFICATIONS]] "Future: recurring
  templates" for why that matters); database off Neon → back to Supabase (was the original choice
  before a Render↔Supabase cross-region network timeout forced the move to Neon — confirmed via
  Supabase itself being healthy, just the Render-to-Supabase network path being slow; Google Cloud
  Run and Supabase may not have the same issue since they could end up in matching regions).
  Frontend stays on
  Vercel, unchanged. Needs: a Google Cloud account/project, `gcloud` CLI setup, a new
  Supabase project (or reuse the one created 2026-06-26 if it still exists), and updated env vars
  (`Cors__AllowedOrigins__0` stays pointed at the same Vercel URL, only `ConnectionStrings__DefaultConnection`
  and wherever the API is hosted change).

## Infra / data
- [ ] **Migration strategy** — there are no EF migrations; `EnsureCreated` + auto drop+recreate **wipes data on schema drift**. Fine for a local/throwaway DB; production needs real migrations or an accepted reset policy. See [[ARCHITECTURE]].
- [x] **Dead `fn_*` cleanup** ✅ 2026-06-29 — `DatabaseFunctions.cs` (56 functions, only 7 used) deleted entirely; all logic now in C#/EF Core. See [[2026-06-29-data-layer-migrated-to-csharp]]. (The unused `OnTime_Backend/sql/functions/*.sql` mirror files, never loaded by the app, are still present — low-priority cleanup, not deploy-gating.)

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
