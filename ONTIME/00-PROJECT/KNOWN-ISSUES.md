# Known Issues — Development Backlog

Hub: [[OnTime]] · Order of work: [[ROADMAP]].

Bugs and tech debt to fix during **normal development** — long before deploy is even considered.
This is the catalog; [[ROADMAP]] sets the order. Deploy-gating infra is separate: [[BEFORE-DEPLOY]].

## QA findings 2026-07-02 (full post-deploy pass on live production)

Full sweep requested by the user after the first production deploy: mobile + desktop, dark/light,
i18n, register→client flow, "Como Funciona" accuracy. Found and fixed:

- [x] **`BrandsPage`/`LeadSourcesPage` mobile cards missing all action buttons** — the mobile card
      view (`renderMobileCard`, swapped in below 768px) only rendered name + status tag, no Edit
      or vehicle-brands buttons at all. Copy-paste gap vs `VehiclesPage.tsx`, which did this
      correctly. Fixed 2026-07-01, confirmed live in production at 375-430px viewport.
- [x] **Redundant `BottomNav`** — a fixed bottom tab bar on mobile duplicated the hamburger
      drawer's nav (which already lists everything). Removed entirely 2026-07-01.
- [x] **"Como Funciona" described the old hardcoded temperature logic only** — the
      Pipeline/Notifications sections said temperature always recalculates from a fixed 72h/240h
      rule on every stage change, and didn't mention recurring notification templates or the
      pg_cron automation at all — both shipped this sprint (see
      [[../04-DECISIONS/2026-06-30-stage-driven-temperature-and-notifications]]). Fixed 2026-07-02
      in both `I18nController.cs` (pt-PT + en-US) and `i18nFallback.ts`.

**Verified working, no changes needed:** desktop table view (1920px, no regression from the mobile
card fix), dark/light mode toggle, i18n key parity (534/534 en-US vs pt-PT, checked live against
`GET /api/i18n`), end-to-end register→create-Company→create-Stand→create-Client flow on a
brand-new test account, RLS enabled project-wide on Supabase with zero app impact (app's DB role
owns every table).

**Not independently re-verified this pass** (deliberately skipped, not a known gap): sending an
extra test email via the internal cron endpoint — would have required embedding the production
`InternalJobs:SecretKey` in a browser-executed script, which the session's own safety tooling
correctly blocked as a credential-exposure risk. The email-send path was already proven for real
in the previous session (a live Business Summary email was received). Language-switcher UI
interaction (the antd `Select` in Profile) could not be reliably automated via the browser tool in
this pass — worth a manual click-through if anyone doubts it; the underlying data path (locale
saved to `User.Locale`, `/api/i18n?locale=X` serving full parity) is confirmed correct via direct
API calls.

## Backend bugs
- [x] **Trial lockout** — fixed T2 (2026-06-06): `SubscriptionAccessMiddleware` now reads `TrialEndsAt` alongside `AccountStatus`; `PendingActivation + TrialEndsAt > now` is treated as full access.
- [x] **Admin gets salesperson view on `/api/clients`** — fixed T1 (2026-06-06): `AccessScope.ManagerBrandScope` now uses `IsManagerOrAdmin` (role≥1), so Admin (role=2) correctly gets brand-wide results.
- [x] **`?? Guid.Empty` claim fallback masks errors** — fixed T1 (2026-06-06): `RequireCompanyId()` / `RequireBrandId()` / `ManagerBrandScope` now throw `AUTH_FORBIDDEN` when the required claim is missing. All 9 sites removed.
- [x] **`SubscriptionService.GetStatusAsync` returns wrong shape** — fixed T3 (2026-06-06): now reads `SubscriptionStatus`, `Plan`, `TrialEndsAt`, `SubscriptionExpiresAt` from DB; `IsTrialActive` and `DaysUntilExpiry` computed correctly.
- [x] **`AuthService.LoginAsync` IsActive logic is confused** — fixed T4 (2026-06-06): simplified to `if (!user.IsActive) throw USER_INACTIVE`. All soft-deleted users are rejected at login regardless of AccountStatus.
- [x] **`PermissionService` collapses 4 flags into 2 on update** — fixed T5 (2026-06-06): `MenuPermissionDto` and `UpdateMenuPermissionRequest` now expose all 4 flags; `UpdatePermissionsAsync` stores each independently.
- [x] **Proposal can be created with no vehicle** — this entry was stale; already fixed (T14, see [[ROADMAP]] Phase 0): both `ClientService.CreateAsync` and `ProposalService.CreateForClientAsync` throw `PROPOSAL_MISSING_VEHICLE` (422) when no vehicle is provided. Verified in source 2026-06-27.
- [ ] **No version/colour seed data** — `SeedVehicleBrandsAsync` seeds brands + models only. `GET /api/vehicles/models/{id}/versions` returns 200 + empty; exterior/interior colour pickers are empty. Either seed versions+colours for active models or surface model-config UI. See [[VEHICLES]].
- [x] **Missing i18n keys** — fixed T12 (2026-06-24): `i18nStore` now merges the API map with `PT_PT_FALLBACK`, so `ACTION.PROPOSAL.LOST`, `MSG.CONVERT.SOLD_AT_HINT`, and `ACTION.SUBSCRIPTION.ACTIVATE` no longer render as raw keys even when the backend response is partial. See [[I18N]], [[FRONTEND-ISSUES]].
- [x] **`ProtectedRoute managerOnly` blocked Admin** — fixed 2026-06-25: checked `role !== 1` only, so Admin (role=2) could see Marcas/Admin/Controlo de Acesso in the sidebar (after an earlier session's fix there) but got bounced straight back to `/dashboard` the instant they clicked — the route guard and the menu-visibility check had drifted out of sync. Now `role !== 1 && role !== 2`.
- [x] **Free-text vehicle proposal showed `vehicleName: " "`** — fixed 2026-06-27: root cause was `fn_get_proposals_paged`'s catalog-name subquery using `LEFT JOIN`s, so a free-text-only vehicle (no `model_id`) produced `CONCAT(NULL, ' ', NULL)` = `" "` (Postgres treats `CONCAT`'s NULL args as empty string, not NULL overall) — `COALESCE` saw a non-NULL `" "` and never fell through to the free-text branch. Changed the subquery's joins to inner `JOIN`s so it returns no row (true `NULL`) when there's no catalog model, letting `COALESCE` fall through correctly. Regression test added (`ProposalSaleFlowTests`).
- [x] **`SendRequestAsync` returned the wrong person's name/email** — fixed 2026-06-25: `FriendshipService.SendRequestAsync` built the response DTO with `receiver.Email` in the `SenderEmail` field and an empty `SenderName`. No test ever asserted the response body's field values, only the status code — exactly the kind of gap [[TESTING]] now calls out. Regression test added (`FriendsFlowTests`).
- [x] **`/api/admin/*` was reachable by any Manager, not just platform Admin** — fixed 2026-06-25, see [[SECURITY]] Authorization section. Real cross-tenant data leak (list/disable/edit other companies), not a single-company permission gap. New `AdminOnly` policy + `AdminFlowTests`.
- [x] **Goal progress used "today" instead of "start of period"** — originally fixed 2026-06-25 by defaulting `startDate` to `startOfPeriod(period)` instead of today. Superseded 2026-06-30: `StartDate`/`EndDate` were removed from `UserGoal` entirely — there's no stored date at all anymore, progress is always computed against the *current* calendar period (`UserGoalService.CurrentPeriodWindow`, based on `DateTimeOffset.UtcNow`), so this whole class of bug can no longer occur. See [[STATUS]], [[SCHEMA-CONFIG]].

## Tech debt
- [ ] **Dual data layer** — complex write flows live in C# services while ~43 `fn_*` (incl. drifted `fn_register_manager`) sit unused in `DatabaseFunctions.cs`. Decide each complex flow's home (SP vs service) per [[2026-05-30-data-layer]], then delete the duplicates so there's one implementation.
- [ ] **i18n is hardcoded** in `I18nController.cs`; `TranslationEntry` table is dead. Decide keep-hardcoded vs move-to-DB. See [[I18N]]. **Note (2026-06-25): the hardcoding itself is no longer a correctness bug** — pt-PT and en-US are now both genuinely complete and kept in sync (see [[I18N]] for the validation checklist) — this entry is purely about the architectural choice (DB-backed vs compiled dictionary), not missing/wrong translations.
- [ ] **Dashboard makes 4 separate DB round-trips** (`fn_get_dashboard_kpis`, `fn_get_monthly_stats`, `fn_get_loss_reasons`, `fn_get_hot_deals` in `SaleRepository`) — found during a Supabase query audit 2026-06-25. Fine on local Postgres; on a shared/pgbouncer-limited Supabase tier, 4 round-trips per dashboard load adds up under concurrent users. Combine into one aggregating function if this becomes a bottleneck.
- [ ] **`FriendshipRepository.SearchUsersAsync` re-queries friendships separately** from the main user search (2 queries where a single join/correlated-subquery would do) — same audit, not yet fixed, low priority (search results are capped at 10 rows).
- [x] **No CI/CD** — fixed 2026-06-27: GitHub Actions in both repos run `dotnet test`/`npm run build` on every push/PR to `main`. No branch-protection rule yet (merge isn't blocked by a red check) — see [[CI-CD-WORKFLOW]].
- [ ] **No structured logging** — default `ILogger` only. Add Serilog with at least a request-id enricher before prod.
- [ ] **No HTTPS redirection / HSTS** in `Program.cs` — fine behind Google Cloud Run's TLS terminator, but explicit middleware is safer.
- [x] **No request rate limiting** — login fixed 2026-06-25 (see Security hardening below). General/DoS-wide rate limiting still not in place.
- [ ] **`AppDbContext` implements both `IAppDbContext` and `IUnitOfWork`** — convenient but blurs the seams. Either document the choice or split.
- [ ] **`ProposalsController` routing is anomalous** — no `[Route]` on the class, full paths on each action. Inconsistent with siblings.
- [ ] **`AuthService` ignores `Subscription:TrialDays` config** — `TrialEndsAt` is hardcoded to `+14 days` in both `RegisterManagerAsync` and `RegisterSalespersonAsync`. The config key exists but is never injected. Fix: inject `IOptions<SubscriptionSettings>` and use `TrialDays`. Also add a test helper `SetTrialEndsAt` to make trial state explicit in tests rather than relying on config.
- [ ] **Test config sets `TrialDays=0, GracePeriodDays=0`** — has no effect because AuthService ignores the config (see above). Once the config is wired, this will matter.
- [x] **Schema sentinel column is fragile** — fixed 2026-06-25: `DatabaseInitializer.IsSchemaCurrentAsync` now checks every table AND every column the current EF model expects against `information_schema`, not one hardcoded column. Found because adding `UserGoal.ShowOnDashboard` reproduced exactly the failure mode this was warning about (500 on insert, old check said "schema current"). **Side effect when this first ran**: it found a large amount of real, long-accumulated drift between the model and the persistent local dev DB (EnsureCreated never retrofits columns onto existing tables) and triggered the documented drop+recreate — wiping local dev data once. Expected/accepted per the no-migrations approach, not a bug, but worth knowing if your local data disappears after pulling this change.
- [x] **Drift detector didn't check indexes** — fixed 2026-06-25: the table/column check above still missed a whole category — adding `HasIndex(...)` to an entity already deployed never reached an existing DB (`EnsureCreated` only creates, never alters). `IsSchemaCurrentAsync` now also diffs `pg_indexes` against every index the EF model expects. Found while adding a `Sale.SoldAt` index that silently never applied.
- [x] **No persisted record of error responses** — added 2026-06-25: `ErrorLog` entity + `ErrorHandlingMiddleware` now writes one row per error (4xx/409/500) to `error_logs`. `GET /api/admin/error-logs` (`AdminOnly`, paginated) to read them. See [[SECURITY]].

## Test-quality findings (2026-06-25 audit)
Most "cross-tenant" suspects (Goals, Notifications, Stages, NotificationPreferences) turned out
to be **already correctly protected** in the service layer (ownership checks like
`goal.UserId != userId`) — just never asserted by a test, so a future refactor regression
wouldn't be caught. Added explicit rejection tests for all of them (`GoalFlowTests`,
`NotificationFlowTests`, `StageConfigFlowTests`). The one *actual* live gap was the Admin
panel above — not a pattern, an outlier. Remaining gap: most `ManagerOnly` controllers
(Brands, Users, Vehicles, Permissions) still have no test confirming a Salesperson gets 403 —
the policy IS enforced, just not regression-tested.

## Security hardening (see [[SECURITY]])
- [x] **PBKDF2 only 10,000 iterations** — fixed 2026-06-25: raised to 600k (OWASP 2023), iteration count embedded per-hash so existing passwords keep verifying.
- [x] **No login rate limiting** — fixed 2026-06-25: 5/min per IP via ASP.NET 8 `AddRateLimiter`.
- [ ] Consider shorter JWT expiry + refresh token (currently 8h, no revocation).
- [ ] Make role claim mapping explicit (emitted `"role"`, read as `ClaimTypes.Role` via default mapping — can silently break).
- [ ] **`/api/friends/search`** has no rate limit of its own (min query length + email masking added 2026-06-25, but still ≤10-row lookups with no throttle). Low priority — revisit if abuse is observed.

## Frontend bugs
Catalogued with file refs in [[FRONTEND-ISSUES]]. Each one is a task in [[AI-TASK-PLAN]] (T6–T9).

## Related
[[ROADMAP]] · [[SECURITY]] · [[FRONTEND-ISSUES]] · [[BEFORE-DEPLOY]] · [[2026-05-30-data-layer]] · [[OnTime]]
