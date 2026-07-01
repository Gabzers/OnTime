# Security

Hub: [[OnTime]] · Architecture: [[ARCHITECTURE]] · Verified against source 2026-06-25.

## Authentication
- **JWT Bearer**, HMAC-SHA256, **8h** expiry (`Jwt:ExpirationMinutes=480`). Stateless — logout = client discards the token (no server revocation).
- Validation (Program.cs): issuer, audience, lifetime, signing key all checked.
- Claims: `NameIdentifier`=userId, `Email`, `role`, `cid`=companyId, `bid`=brandId, `jti`.
- **Note:** token emits role as `"role"` but the policy/middleware read `ClaimTypes.Role` — relies on the default JWT inbound claim mapping. If `MapInboundClaims=false` is ever set, role checks silently break.

## Passwords
- **PBKDF2-SHA256, 600,000 iterations** (raised 2026-06-25, was 10,000 — OWASP 2023 guidance), 16-byte random salt, 32-byte hash (`Pbkdf2PasswordHasher`).
- Stored as `iterations.base64(salt).base64(hash)` — the iteration count is embedded per-hash so raising it again later won't invalidate existing passwords. Legacy 2-part hashes (no embedded count, from before this format) fall back to 10,000 on verify — backward compatible, verified against the seeded demo admin account after the change.
- Verify uses `CryptographicOperations.FixedTimeEquals` (constant-time) ✅.

## Authorization
- `[Authorize]` is the default on all controllers; `[AllowAnonymous]` only on auth + i18n.
- **`ManagerOnly` policy** = role `1` (Manager) OR `2` (Admin) — for endpoints scoped to *the caller's own* company (Brands, Vehicles, Users, Permissions).
- **`AdminOnly` policy** = role `2` only — added 2026-06-25 after `AdminController` (`/api/admin/*`) was found requiring only `ManagerOnly`, meaning **any paying customer's Manager could list, disable, or edit every other company's data** (cross-tenant leak, not a single-company permission gap). `AdminController` and `ErrorLogsController` (see [[KNOWN-ISSUES]]/error logging) both use `AdminOnly`. Rule of thumb: if an endpoint can read/write a company OTHER than the caller's own, it must be `AdminOnly`, never `ManagerOnly` — regression test: `AdminFlowTests.cs`.
- **Admin (role=2) bypasses** the subscription middleware AND permission checks (`PermissionService` returns full access computed in-memory). Powerful — admin creation must stay controlled (see AdminBootstrap below).
- `IsManager()` helper = role `== 1` only (Admin is not "manager" for brand-scoping in `ClientsController`).
- The per-company configurable menu-permission set (`PermissionService.AllRoutes`, surfaced in Access Control) deliberately excludes `/admin` — cross-tenant access is a platform-level constant, never a tenant-configurable permission. A one-time DB cleanup in `DatabaseInitializer` deletes any `/admin` row seeded before this fix.

## Multi-tenant isolation
- Every service verifies ownership: `client.UserId != userId` → `CLIENT_WRONG_USER`; goals/friends → `AUTH_FORBIDDEN`.
- Managers are scoped to their `brandId`; salespeople to their own `userId`.
- Read stored functions filter by `user_id` / `brand_id` in SQL.
- **Privacy:** `commission` is never returned in friend-facing endpoints; public profile fields gated by `UserPublicProfile.Show*` flags. See [[FRIENDS]].

## Account/subscription gating
`SubscriptionAccessMiddleware` blocks by `AccountStatus` (table in [[ARCHITECTURE]]). UI permissions are advisory only — real enforcement is the attributes above. See [[GOALS-PERMISSIONS]].

## Injection
- Read SPs are called via `db.Database.SqlQuery<T>($"… {userId} …")` — the interpolated `FormattableString` is **parameterized** by EF/Npgsql, not string-concatenated. Safe.
- Function bodies are constant strings applied at startup.

## Secrets
- ✅ **Not committed.** `appsettings.Development.json` and `.env` are gitignored and untracked. Base `appsettings.json` ships with empty `Jwt:Key`, connection string, Stripe/Ifthenpay keys, and AdminBootstrap.
- Prod must supply these via environment/config. See [[BEFORE-DEPLOY]].
- **AdminBootstrap**: creates a demo admin only if (a) no companies exist AND (b) email+password are configured. Prod base config leaves them empty → skipped. The dev file has `admin@ontimecrm.io / Admin123!` for **local only**.
- `.env.example` had a real, weak literal (`ADMIN_PASSWORD=teste`) instead of a placeholder like the other secrets — fixed to `REPLACE_WITH_A_STRONG_PASSWORD` (2026-06-25). If anyone ever copies `.env.example` straight into prod env vars without changing this, AdminBootstrap would create a real admin with that password — **verify Cloud Run's env vars don't have this value** before going live.

## Login hygiene
- Login returns generic `USER_INVALID_CREDENTIALS` for both unknown email and wrong password → no user enumeration. ✅
- Register returns `USER_EMAIL_TAKEN` (enumeration on register — common tradeoff, acceptable for MVP).
- **Rate limited** (added 2026-06-25): `POST /api/auth/login` is capped at 5 requests/minute per client IP (`[EnableRateLimiting("login")]`, ASP.NET 8 built-in `AddRateLimiter` + `RateLimitPartition.GetFixedWindowLimiter`), 429 beyond that. Partitioned by IP, not a single shared counter, so one attacker can't lock out other users. The limit is configurable via `RateLimiting:LoginPermitPerMinute` — `TestWebAppFactory` sets it very high (100000) because the whole integration suite shares one TestServer/IP through many sequential logins and would otherwise trip the production limit on itself.
- Verified manually: 5×401 then 429 on repeated bad-credential attempts against the running stack.

## Friends search — cross-tenant by design, hardened 2026-06-25
`GET /api/friends/search` is intentionally **not** scoped to company/brand (the Friends feature is
a cross-dealership social layer — see [[FRIENDS]]), but an unscoped, platform-wide user directory
search is a scraping target. Hardened:
- Minimum 2-character query (was: any non-empty string, including a single letter).
- Email is masked in results (`j***@example.com`) — only revealed in full once two users are
  actually friends (existing `FriendRequestDto.SenderEmail`/profile flows, unchanged).
No rate limit on `/search` itself yet — low priority since it returns ≤10 rows per call and
requires authentication, but revisit if abuse is observed.

## Error logging (added 2026-06-25)
`ErrorHandlingMiddleware` persists every error response (ApiException 4xx, DB conflict 409,
unhandled 500 with stack trace) to the `error_logs` table — status, code, message, path, method,
trace ID, and the caller's userId if authenticated. `GET /api/admin/error-logs` (paginated,
`AdminOnly`) is the only way to read them — never exposed to a Manager. Logging failures are
swallowed (after going through `ILogger`) so they can never replace or mask the real error
response; the scoped `DbContext`'s change tracker is cleared first so a failed save (e.g. the
409 case) doesn't get re-attempted alongside the new log row.

## Swagger/API surface audit (2026-07-01, post-Cloud-Run-deploy)
- [ ] **Swagger/Scalar exposed unconditionally in Production** — `Program.cs` calls
      `app.UseSwagger()`/`UseSwaggerUI()`/`MapScalarApiReference()` with no
      `if (app.Environment.IsDevelopment())` gate, so the full API surface (every route, DTO shape,
      including `/api/internal/run-scheduled-jobs`) is publicly browsable at the live Cloud Run URL.
      Helps an attacker's reconnaissance for free. Fix: gate behind `IsDevelopment()`, or put it
      behind its own auth in prod if API docs need to stay reachable for integration work.
- [ ] **`POST /api/internal/run-scheduled-jobs` has no rate limiting** — guarded only by a shared
      secret header (`X-Internal-Key`, no `[Authorize]` at all since there's no logged-in user
      behind a cron call). The secret itself is strong (64 random bytes, stored in Supabase Vault,
      never in git — see [[BEFORE-DEPLOY]]), but there's no throttling or alerting on repeated wrong
      guesses the way `/login` has. Low urgency given key strength, but cheap to add for defense in
      depth once other Sprint 2 hardening lands.
- [ ] **`POST /api/auth/register` and `/register-manager` have no rate limiting** — only `/login`
      does today. Allows mass account-creation spam, and (combined with the existing
      `USER_EMAIL_TAKEN` enumeration tradeoff noted above) makes email-enumeration-at-scale cheaper
      than it should be. Same `[EnableRateLimiting]` + `RateLimitPartition` pattern as login would
      close this.
- [ ] **`GET /api/auth/companies` / `companies/{id}/brands` are `[AllowAnonymous]` and unscoped** —
      needed for the registration screen's company/Filial picker, but also lets anyone (no login
      required) enumerate every customer company and Filial name in the system. Low severity
      (business-directory leak, not personal data), but worth a second look before onboarding
      competitor-sensitive customers — could require Manager approval only, or paginate/rate-limit.

## Hardening backlog (dev-time) → [[KNOWN-ISSUES]]
Consider shorter JWT + refresh token (currently 8h, no revocation) · rate limit `/friends/search`.
## Prod config gating → [[BEFORE-DEPLOY]]
Lock CORS (no `AllowAnyOrigin` fallback) · `AllowedHosts` · real secrets · don't ship the demo admin.

## Related
[[ARCHITECTURE]] · [[GOALS-PERMISSIONS]] · [[FRIENDS]] · [[KNOWN-ISSUES]] · [[BEFORE-DEPLOY]] · [[OnTime]]
