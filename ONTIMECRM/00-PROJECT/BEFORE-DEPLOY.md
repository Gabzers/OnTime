# Before Deploy — Checklist

Hub: [[OnTimeCRM]] · Roadmap: [[ROADMAP]].

**Only genuine deploy-gating infra/config lives here** — things that must be right to ship to Render/Vercel/Supabase, regardless of feature work.

> Bugs and tech debt are NOT here — they're normal development, done well before deploy is considered. See [[KNOWN-ISSUES]]. All of those should already be resolved by the time this checklist matters.

## Infra / data
- [ ] **Migration strategy** — there are no EF migrations; `EnsureCreated` + auto drop+recreate **wipes data on schema drift**. Fine for a local/throwaway DB; production needs real migrations or an accepted reset policy. See [[ARCHITECTURE]].
- [ ] **Dead `fn_*` cleanup** — the ~43 unused/drifted stored functions are re-applied on every startup. Harmless locally; remove before prod once the data-layer reconciliation in [[KNOWN-ISSUES]] is done.

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
[[KNOWN-ISSUES]] · [[ROADMAP]] · [[ARCHITECTURE]] · [[PAYMENTS]] · [[OnTimeCRM]]
