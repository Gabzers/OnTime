# AI Task Plan — Step by Step (test-first)

Hub: [[OnTimeCRM]] · Loop & gate: [[AI-WORKFLOW]] · Backlog: [[KNOWN-ISSUES]].

Ordered tasks for the local AI. **Every task runs the [[AI-WORKFLOW]] loop**: RED → GREEN →
verify everything → update docs + session log. Top to bottom; don't start the next until the
gate is green.

---

## T0 — Frontend test harness (enabler) ✅ 2026-06-06
- **Do:** add Vitest + React Testing Library + jsdom to `OnTimeCRM_Frontend`; `test` script; one trivial sample test.
- **Done when:** `npm run test` runs and the sample passes; `npm run build` still green.
- No behavior change. Unblocks every frontend task below.

## T1 — Access scope helper ⭐ (enabler — fixes T2/T3 in one shot) ✅ 2026-06-06
- **Decision first:** accept [[2026-05-30-access-scope-helper]] (or modify).
- **RED:** two xUnit tests in a new `AccessScopeFlowTests`:
  - Admin (role=2) hitting `GET /api/clients` sees **brand-wide** results (today: own-only).
  - A manager request with the `bid` claim removed → **401/403**, not 200 with 0 rows.
- **GREEN:** add `IsAdmin()`, `IsManagerOrAdmin()` to `ClaimsPrincipalExtensions`; introduce `AccessScope` value type + `User.Scope()`. Sweep `ClientsController`, `BrandsController`, `UsersController` (9 sites of `?? Guid.Empty` to remove). Throw `AUTH_FORBIDDEN` when a required claim is missing.
- **Done when:** both tests green AND all existing flow tests still green.
- **Docs:** mark the decision as accepted; tick the two related bugs in [[KNOWN-ISSUES]].

## T2 — Trial access (backend) ✅ 2026-06-06
- **RED:** `TrialAccessFlowTests` — 3 tests (active trial → 200 GET/POST; expired trial → 402).
- **GREEN:** `SubscriptionAccessMiddleware` reads `TrialEndsAt`; `PendingActivation + TrialEndsAt > now` → full access.
- **Done:** all 3 new tests + all 11 SubscriptionFlowTests green. Test 4 in SubscriptionFlowTests updated (was encoding the bug — now manually expires trial before asserting 402).
- **Note:** `AuthService` hardcodes 14 days — `Subscription:TrialDays` config is never read. See KNOWN-ISSUES tech debt.

## T3 — Subscription status DTO (backend) ✅ 2026-06-06
- **RED:** 2 new tests in `SubscriptionFlowTests` (Tests 10+11): after registration `SubscriptionStatus=Trial, Plan=Trial, IsTrialActive=true`; after activation `SubscriptionStatus=Active`.
- **GREEN:** `SubscriptionService.GetStatusAsync` now reads `SubscriptionStatus`, `Plan`, `TrialEndsAt`, `SubscriptionExpiresAt` from DB. `IsTrialActive` and `DaysUntilExpiry` computed correctly. All 13 subscription tests green.

## T4 — Login `IsActive` logic (backend) ✅ 2026-06-06
- **RED:** Test 4b in `AuthFlowTests` — `IsActive=false + AccountStatus=Inactive` → expected 401/USER_INACTIVE, got 200 (bug confirmed).
- **GREEN:** `AuthService.LoginAsync` simplified to `if (!user.IsActive) throw USER_INACTIVE`. All 8 auth tests green.

## T5 — Permission flags expose all 4 (backend) ✅ 2026-06-06
- **Decision:** expose all 4 flags (`CanView/Create/Edit/Delete`) — entity already had them, collapse was an oversight.
- **RED:** `PermissionFlowTests` — 3 tests; update roundtrip with independent flags failed (they all became `CanEdit`).
- **GREEN:** expanded `MenuPermissionDto` + `UpdateMenuPermissionRequest` to 4 flags; removed synthetic collapse in `PermissionService.UpdatePermissionsAsync`; fixed `ToDto` and admin computed set. Fixed `GoalFlowTests` reference to old `CanRead` field → `CanView`. Also fixed `SubscriptionFlowTests.InactiveAccount` (was relying on T4 bug; updated to get token before deactivation). All 99 tests pass except 7 pre-existing failures.

---

## Frontend bug sweep (each is one TDD cycle)

## T6 — Snooze field name ✅ 2026-06-06
- **RED:** Vitest in `src/__tests__/api/notifications.test.ts` — confirmed body key was `snoozeUntil` not `snoozedUntil`.
- **GREEN:** renamed `SnoozeNotificationRequest.snoozeUntil` → `snoozedUntil` in `types/api.ts`; fixed callers in `NotificationDropdown.tsx` and `NotificationsPage.tsx`. Build + tests green.

## T7 — Notification status labels (full-stack) ✅ 2026-06-06
- **RED:** 4 backend Theory tests (pt-PT + en-US × .2/.3) + 2 Vitest tests on `PT_PT_FALLBACK` — all failed (labels swapped).
- **GREEN:** swapped `.2`/`.3` in `I18nController` (both locales) + `i18nFallback.ts`; fixed colors in `NotificationStatusTag` (`2→blue, 3→default`) and `NotificationDropdown.statusColor` (`status===2` for snoozed); corrected comment in `types/api.ts`. 10 backend + 4 frontend tests green.

## T8 — `dueAt` → `scheduledFor` ✅ 2026-06-06
- **RED:** `tsc -p tsconfig.app.json --noEmit` — test assigning `scheduledFor` to `NotificationDto` produced TS2353/TS2339 errors (field didn't exist on the type).
- **GREEN:** renamed `NotificationDto.dueAt` → `scheduledFor`; `CreateNotificationRequest.dueAt` → `scheduledFor`; also corrected `notes` → `body` (backend DTO has `Body`); added `doneAt?` that was missing. Fixed `NotificationsPage.tsx` render. TypeScript clean, build green.

## T9 — Hardcoded strings + `brandColor` typing ✅ 2026-06-06
- **RED:** sentinel-mock test on `NotificationDropdown` — found hardcoded "Ver todas" bypassing i18n.
- **GREEN (full sweep):**
  - `NotificationDropdown`: `title="Adiar 1h"` → `t['ACTION.NOTIFICATION.SNOOZE_1H']`; `"Ver todas"` → `t['ACTION.VIEW_ALL']`
  - `api/axios.ts` 403 handler: reads `MSG.ERROR.FORBIDDEN` from i18nStore instead of hardcoded string
  - `App.tsx`: `(user as any)?.brandColor` → `user?.brandColor` (type already existed)
  - `usePermissions.ts`: `(user as any)?.role` → `user?.role`; fallback updated to 4-flag shape
  - `types/api.ts`: `MenuPermissionDto` + `UpdateMenuPermissionRequest` updated to `canView/canCreate/canEdit/canDelete` (missed in T5)
  - `AccessControlPage.tsx`: expanded from 2 to 4 permission columns
  - Backend + frontend i18n: added `ACTION.NOTIFICATION.SNOOZE_1H`, `ACTION.VIEW_ALL`, `MSG.ADMIN.FULL_ACCESS`, `LABEL.PERMISSION.CAN_VIEW/CREATE/DELETE`, en-US `MSG.ERROR.*` and notification action entries that were missing.
  - 6 frontend tests + TypeScript clean + build green. Backend 96/103 (7 pre-existing).

---

## Reported bugs — June 2026 (each one TDD cycle; details in [[FRONTEND-ISSUES]])
- **T10** ✅ 2026-06-06 — Theme flash (FOUC): inline `<script>` in `index.html` reads `ontimecrm-theme` from localStorage and applies `.dark` class before React mounts; `html.dark body` CSS sets `background-color:#141414`. 4 Vitest tests green. `setupTests.ts` added (localStorage + matchMedia mocks).
- **T11** ✅ 2026-06-06 — Encoding verified non-issue: source is correct UTF-8, ASP.NET Core 8 sends `charset=utf-8` automatically. 2 new backend tests confirm. Added `.editorconfig` (`charset=utf-8-bom` for `.cs`) as preventive measure.
- **T12** ✅ 2026-06-06 — 4 missing i18n keys added (`ACTION.PROPOSAL.LOST`, `ACTION.SUBSCRIPTION.ACTIVATE`, `MSG.CONVERT.SOLD_AT_HINT`, `MSG.SALE.CREATED`) — both pt-PT and en-US in `I18nController.cs` and `i18nFallback.ts`. 8 backend Theory tests green (116 total).
- **T13** ✅ 2026-06-06 — `ProposalDetailPage` back button reads `location.state.fromClientId/fromTab` and navigates to `/clients/detail` with `{ clientId, tab }` if present, else `navigate(-1)`. `ClientDetailPage` reads `location.state.tab` for `defaultActiveKey`; passes `fromClientId/fromTab` state when navigating to proposals. 2 Vitest tests green (12 total).
- **T14** — Require ≥1 vehicle at proposal creation (backend) so convert can't later 422. See [[VEHICLES]].
- **T15** — Vehicles screen: status dot (grey/red/green) + active/inactive toggle + `PATCH /models/{id}/active`. See [[VEHICLES]].

## Phase 1+ (same loop, larger units)
- **F1 — User Brands filter** ([[USER-BRANDS]]): `UserVehicleBrand` join + profile multi-select + filter vehicle lists/pickers by the user's brands. Confirm the open decisions in that note first.
- **Seed versions/colours** for active models (or model-config UI) so proposals pick a real configuration ([[VEHICLES]]).
- **Data-layer reconciliation** ([[2026-05-30-data-layer]]): decide each complex flow's home, delete the drifted duplicates. Guard: tests stay green; add tests where coverage is missing.
- **Audit Goals / Friends / Permissions** end-to-end: one flow test per happy path + ownership/403; fix wiring gaps. See [[GOALS-PERMISSIONS]], [[FRIENDS]].
- **Hygiene mini-track**: CI (GitHub Actions running `dotnet test` + `npm run build`) → Serilog with request-id → ASP.NET rate limiting → HTTPS/HSTS middleware. Each is its own TDD-ish task.
- **Payments** ([[PAYMENTS]]): TDD against existing WireMock stubs — flow test "initiate → webhook → account Active", then implement Stripe/Ifthenpay clients.
- **i18n rebuild** ([[I18N]]): `/grill-me` first to decide hardcoded vs DB, then test-first.

Order and detail for these live in [[ROADMAP]]; expand each into T-style tasks when you reach them.

## Related
[[AI-WORKFLOW]] · [[ROADMAP]] · [[KNOWN-ISSUES]] · [[TESTING]] · [[2026-05-30-access-scope-helper]] · [[OnTimeCRM]]
