# Security

Hub: [[OnTimeCRM]] · Architecture: [[ARCHITECTURE]] · Verified against source 2026-05-30.

## Authentication
- **JWT Bearer**, HMAC-SHA256, **8h** expiry (`Jwt:ExpirationMinutes=480`). Stateless — logout = client discards the token (no server revocation).
- Validation (Program.cs): issuer, audience, lifetime, signing key all checked.
- Claims: `NameIdentifier`=userId, `Email`, `role`, `cid`=companyId, `bid`=brandId, `jti`.
- **Note:** token emits role as `"role"` but the policy/middleware read `ClaimTypes.Role` — relies on the default JWT inbound claim mapping. If `MapInboundClaims=false` is ever set, role checks silently break.

## Passwords
- **PBKDF2-SHA256, 10,000 iterations**, 16-byte random salt, 32-byte hash (`Pbkdf2PasswordHasher`).
- Stored as `base64(salt).base64(hash)`. Verify uses `CryptographicOperations.FixedTimeEquals` (constant-time) ✅.
- ⚠️ 10k iterations is well below OWASP 2023 guidance (~600k for PBKDF2-SHA256). Hardening item → [[KNOWN-ISSUES]].

## Authorization
- `[Authorize]` is the default on all controllers; `[AllowAnonymous]` only on auth + i18n.
- **`ManagerOnly` policy** = role `1` (Manager) OR `2` (Admin).
- **Admin (role=2) bypasses** the subscription middleware AND permission checks (`PermissionService` returns full access computed in-memory). Powerful — admin creation must stay controlled (see AdminBootstrap below).
- `IsManager()` helper = role `== 1` only (Admin is not "manager" for brand-scoping in `ClientsController`).

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

## Login hygiene
- Login returns generic `USER_INVALID_CREDENTIALS` for both unknown email and wrong password → no user enumeration. ✅
- Register returns `USER_EMAIL_TAKEN` (enumeration on register — common tradeoff, acceptable for MVP).
- ⚠️ No rate limiting on login → brute-force exposure. Hardening item → [[KNOWN-ISSUES]].

## Hardening backlog (dev-time) → [[KNOWN-ISSUES]]
Raise PBKDF2 iterations (or move to Argon2id) · add login rate limiting · consider shorter JWT + refresh token.
## Prod config gating → [[BEFORE-DEPLOY]]
Lock CORS (no `AllowAnyOrigin` fallback) · `AllowedHosts` · real secrets · don't ship the demo admin.

## Related
[[ARCHITECTURE]] · [[GOALS-PERMISSIONS]] · [[FRIENDS]] · [[KNOWN-ISSUES]] · [[BEFORE-DEPLOY]] · [[OnTimeCRM]]
