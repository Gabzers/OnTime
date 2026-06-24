# OnTimeCRM — Project Status

Hub: [[OnTimeCRM]] · Work order: [[ROADMAP]].

Last verified against source: 2026-06-06

## Backend — implemented & wired (controller + service + stored functions)
- [x] Clean architecture (Domain / Application / Infrastructure / API)
- [x] All entities + EF configurations
- [x] ~50 PostgreSQL stored functions (`DatabaseFunctions.cs`)
- [x] `DatabaseInitializer` (EnsureCreated + drift detection + auto-recreate)
- [x] Auth: register manager, register salesperson, login, public company/brand lookup
- [x] Clients: list (manager sees brand), create, hot deals, detail, delete, stage change, history, sales history
- [x] Proposals: list, detail, update, create-for-client, mark lost, convert to sale
- [x] Sales: list, detail, dashboard
- [x] Notifications: list, today, overdue-count, create, done/snooze/ignore (PATCH)
- [x] Notification preferences: get, update (`/api/preferences/notifications`)
- [x] Stages: CRUD, reorder, templates CRUD
- [x] Vehicles: brands, models, versions (manager-only writes)
- [x] Users: me, update me, public profile; manager: list/get/invite/activate/view-dashboard
- [x] **Goals API** — full CRUD, live progress from KPIs (`UserGoalService`)
- [x] **Permissions API** — seed + update, manager-only (`PermissionService`)
- [x] **Friends API** — requests, accept/reject, remove, public profile (`FriendshipService`)
- [x] Subscription: status + cancel work (stub); see below
- [x] Admin: companies + brands CRUD (manager-only)
- [x] Subscription middleware (402/403 enforcement) + error-handling middleware
- [x] Integration tests (xUnit + Testcontainers + Respawn + WireMock). See [[TESTING]]. **103/103 green (2026-06-06)**

## Backend — known bugs / debt
Catalogued in [[KNOWN-ISSUES]] (dev work, not deploy-gating). Highlights: trial lockout (PendingActivation blocks even GET), dual data layer (~43 dead/drifted `fn_*` vs C# services — see [[2026-05-30-data-layer]]), hardcoded i18n.

## Backend — stubbed / missing
- [ ] **Payment integration** — `SubscriptionService.InitiateAsync` throws `PAYMENT_PENDING`; no Stripe/Ifthenpay client exists. `GetStatus` returns defaults, `Cancel` is a no-op. See [[PAYMENTS]].
- [ ] **i18n is hardcoded** in `I18nController.cs` (pt-PT + en-US dictionaries). `TranslationEntry` table exists but is **never read/written** — dead. See [[I18N]].
- [ ] Email verification flow
- [ ] Webhook endpoints for payment confirmation

## Frontend — exists, has bugs
- [x] All pages, routing (lazy), auth flow, Zustand stores, TanStack Query, generic components, API layer, i18n
- [x] **Test harness** — Vitest + React Testing Library + jsdom wired; `npm run test` green (T0 done 2026-06-06)
- [x] **T6–T9 frontend bugs fixed (2026-06-06):** snooze field renamed, enum labels corrected, hardcoded PT strings moved to i18n, `brandColor`/`role` `as any` casts removed, `dueAt→scheduledFor`, `MenuPermissionDto` expanded to 4 flags

## Deployment
- [ ] Not deployed. Local docker-compose only. Targets: Render.com + Vercel + Supabase.
- See [[BEFORE-DEPLOY]] for the must-fix-before-prod checklist (data layer, no-migrations data loss, trial bug, CORS, secrets).

## Pricing
- Monthly 15€/user · Annual 149€/user (~17% off) · 14-day free trial, no card.

---

## Related
[[ROADMAP]] · [[BEFORE-DEPLOY]] · [[FRONTEND-ISSUES]] · [[PAYMENTS]] · [[I18N]] · [[OnTimeCRM]]
