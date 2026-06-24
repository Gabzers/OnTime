# Session — T1: Access scope helper

**Date:** 2026-06-06
**Task:** T1 from [[AI-TASK-PLAN]]

## Decision
Accepted [[2026-05-30-access-scope-helper]] as written.

## What changed

### New: `AccessScope` value type + helpers (`ClaimsPrincipalExtensions.cs`)
- Added `readonly record struct AccessScope(UserId, BrandId, CompanyId, Role)` with:
  - `IsAdmin` (role==2), `IsManagerOrAdmin` (role>=1)
  - `ManagerBrandScope` → returns BrandId for manager/admin, null for salesperson; throws `AUTH_FORBIDDEN` if BrandId is null for a manager/admin
- Added `User.IsAdmin()`, `User.IsManagerOrAdmin()`, `User.Scope()` extensions
- Added `User.RequireCompanyId()`, `User.RequireBrandId()` — throw `AUTH_FORBIDDEN` instead of returning `Guid.Empty`

### Fixed: `ClientsController.GetPaged`
- Was: `User.IsManager() ? GetBrandId() : null` (role==1 only, so Admin got own-only view)
- Now: `User.Scope().ManagerBrandScope` (role>=1, Admin now gets brand-wide view)

### Fixed: `BrandsController` (5 sites)
- All `User.TryGetCompanyId() ?? Guid.Empty` → `User.RequireCompanyId()`

### Fixed: `UsersController` (4 sites)
- All `User.TryGetBrandId() ?? Guid.Empty` → `User.RequireBrandId()`
- `InviteSalesperson` now uses `RequireBrandId()` + `RequireCompanyId()`

### Fixed: `GoalFlowTests.cs`
- Pre-existing compile error: `MenuPermissionDto.CanView` → `CanRead` (property was renamed but test wasn't updated)

## Tests written (`AccessScopeFlowTests.cs`)
1. `Admin_GetClients_SeesBrandWideResults_NotOwnOnly` — Admin (role=2) sees salesperson's client; before fix: 0 results
2. `Manager_WithMissingBidClaim_Returns403_NotSilentEmptyResult` — Admin token without `bid` claim gets 403; before fix: 200 with 0 rows

### Note on test 2 approach
Crafted JWT tokens (manual construction) didn't bypass the subscription middleware's admin check, even with identical role=2 claim — suspected issue with how JwtSecurityTokenHandler's default inbound claim type map resolves during test. Resolved by using `TestHelpers.LoginAsync` after setting `User.BrandId = null` in DB before re-login; this generates a real JWT without `bid` via the normal auth flow, which correctly bypasses the middleware.

## Gate result
- `dotnet test` — 83 pass / 7 fail (7 pre-existing failures unchanged) ✅
- `npm run test` — 1 pass ✅
- `npm run build` — green ✅

## Docs updated
- [[KNOWN-ISSUES]] — two bugs ticked ✅
- [[AI-TASK-PLAN]] — T1 ticked ✅
- [[2026-05-30-access-scope-helper]] — status → implemented ✅

## Next
T2 — Trial access: `SubscriptionAccessMiddleware` should allow `PendingActivation` users with `TrialEndsAt > now` full access (currently blocks all PendingActivation with 402).
