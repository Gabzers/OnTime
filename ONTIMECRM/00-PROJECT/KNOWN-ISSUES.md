# Known Issues — Development Backlog

Hub: [[OnTimeCRM]] · Order of work: [[ROADMAP]].

Bugs and tech debt to fix during **normal development** — long before deploy is even considered.
This is the catalog; [[ROADMAP]] sets the order. Deploy-gating infra is separate: [[BEFORE-DEPLOY]].

## Backend bugs
- [x] **Trial lockout** — fixed T2 (2026-06-06): `SubscriptionAccessMiddleware` now reads `TrialEndsAt` alongside `AccountStatus`; `PendingActivation + TrialEndsAt > now` is treated as full access.
- [x] **Admin gets salesperson view on `/api/clients`** — fixed T1 (2026-06-06): `AccessScope.ManagerBrandScope` now uses `IsManagerOrAdmin` (role≥1), so Admin (role=2) correctly gets brand-wide results.
- [x] **`?? Guid.Empty` claim fallback masks errors** — fixed T1 (2026-06-06): `RequireCompanyId()` / `RequireBrandId()` / `ManagerBrandScope` now throw `AUTH_FORBIDDEN` when the required claim is missing. All 9 sites removed.
- [x] **`SubscriptionService.GetStatusAsync` returns wrong shape** — fixed T3 (2026-06-06): now reads `SubscriptionStatus`, `Plan`, `TrialEndsAt`, `SubscriptionExpiresAt` from DB; `IsTrialActive` and `DaysUntilExpiry` computed correctly.
- [x] **`AuthService.LoginAsync` IsActive logic is confused** — fixed T4 (2026-06-06): simplified to `if (!user.IsActive) throw USER_INACTIVE`. All soft-deleted users are rejected at login regardless of AccountStatus.
- [x] **`PermissionService` collapses 4 flags into 2 on update** — fixed T5 (2026-06-06): `MenuPermissionDto` and `UpdateMenuPermissionRequest` now expose all 4 flags; `UpdatePermissionsAsync` stores each independently.
- [ ] **Proposal can be created with no vehicle** — `ClientService.CreateAsync` / `ProposalService.CreateForClientAsync` add vehicles only if provided; none is required. Later, convert-to-sale throws `SALE_MISSING_VEHICLE` (422). Fix: require ≥1 vehicle (model or free-text) at proposal creation, and don't offer un-configured/inactive models in the picker (see [[VEHICLES]]).
- [ ] **No version/colour seed data** — `SeedVehicleBrandsAsync` seeds brands + models only. `GET /api/vehicles/models/{id}/versions` returns 200 + empty; exterior/interior colour pickers are empty. Either seed versions+colours for active models or surface model-config UI. See [[VEHICLES]].
- [ ] **Missing i18n keys** — components reference keys not defined in `/api/i18n`: `ACTION.PROPOSAL.LOST`, `MSG.CONVERT.SOLD_AT_HINT` (defined as `HINT.PROPOSAL.SOLD_AT`). Render as raw keys. See [[I18N]], [[FRONTEND-ISSUES]].

## Tech debt
- [ ] **Dual data layer** — complex write flows live in C# services while ~43 `fn_*` (incl. drifted `fn_register_manager`) sit unused in `DatabaseFunctions.cs`. Decide each complex flow's home (SP vs service) per [[2026-05-30-data-layer]], then delete the duplicates so there's one implementation.
- [ ] **i18n is hardcoded** in `I18nController.cs`; `TranslationEntry` table is dead. Decide keep-hardcoded vs move-to-DB. See [[I18N]].
- [ ] **No CI/CD** — `dotnet test` + `npm run build` not run automatically on push. Add GitHub Actions before adding contributors.
- [ ] **No structured logging** — default `ILogger` only. Add Serilog with at least a request-id enricher before prod.
- [ ] **No HTTPS redirection / HSTS** in `Program.cs` — fine behind Render's TLS terminator, but explicit middleware is safer.
- [ ] **No request rate limiting** — both for login (security) and general (DoS). ASP.NET 8 has built-in.
- [ ] **`AppDbContext` implements both `IAppDbContext` and `IUnitOfWork`** — convenient but blurs the seams. Either document the choice or split.
- [ ] **`ProposalsController` routing is anomalous** — no `[Route]` on the class, full paths on each action. Inconsistent with siblings.
- [ ] **`AuthService` ignores `Subscription:TrialDays` config** — `TrialEndsAt` is hardcoded to `+14 days` in both `RegisterManagerAsync` and `RegisterSalespersonAsync`. The config key exists but is never injected. Fix: inject `IOptions<SubscriptionSettings>` and use `TrialDays`. Also add a test helper `SetTrialEndsAt` to make trial state explicit in tests rather than relying on config.
- [ ] **Test config sets `TrialDays=0, GracePeriodDays=0`** — has no effect because AuthService ignores the config (see above). Once the config is wired, this will matter.
- [ ] **Schema sentinel column is fragile** — `notification_preferences.new_client_notification_days_after` is the drift check. Remove it accidentally → drift detection silently breaks. Use a versioned migration table or a generic "has all expected columns" check.

## Security hardening (see [[SECURITY]])
- [ ] **PBKDF2 only 10,000 iterations** — raise to ~600k (OWASP 2023) or move to Argon2id. (`Pbkdf2PasswordHasher`)
- [ ] **No login rate limiting** — add throttling/lockout vs brute force. (`AuthController`)
- [ ] Consider shorter JWT expiry + refresh token (currently 8h, no revocation).
- [ ] Make role claim mapping explicit (emitted `"role"`, read as `ClaimTypes.Role` via default mapping — can silently break).

## Frontend bugs
Catalogued with file refs in [[FRONTEND-ISSUES]]. Each one is a task in [[AI-TASK-PLAN]] (T6–T9).

## Related
[[ROADMAP]] · [[SECURITY]] · [[FRONTEND-ISSUES]] · [[BEFORE-DEPLOY]] · [[2026-05-30-data-layer]] · [[OnTimeCRM]]
