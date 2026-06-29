# Vehicle brands move to per-Filial config + multi-Filial membership

Date: 2026-06-27 · Status: **implemented 2026-06-27** · [[OnTime]]

**Rule:** which car brands a Filial sells is configured once by Manager/Admin
(`BrandVehicleBrand`), not picked per-salesperson. A user can belong to several
companies/filiais (`UserBrandMembership`) and switch which one is active; `User.CompanyId`/
`BrandId` and the JWT's `cid`/`bid` claims stay singular — the "currently active" Filial.

**Why:** shipped earlier the same day as [[2026-06-27-per-user-vehicle-catalog]] (per-user
*ownership* of the vehicle catalog) — but that still left every individual salesperson picking
which car brands they personally sell, which doesn't match how a real dealership works: the
Filial decides what it sells, not each employee independently. Discussing that surfaced the
related gap that a user can only ever belong to one company/Filial, fixed at registration, with no
way for a Manager to move someone or for one person to oversee two stands.

**Implies (do / don't):**
- ✅ `BrandVehicleBrand` (`BrandId, VehicleBrandId`) replaces `UserVehicleBrand` entirely — no
  migration needed (no prod data yet).
- ✅ `VehicleRepository.GetModelsAsync`'s visibility filter checks the calling user's **active**
  Filial against `BrandVehicleBrand`, not a per-user selection.
- ✅ Allowing a brand on a Filial doesn't eagerly clone for every user of that Filial — cloning
  stays lazy, triggered the first time each user actually views/creates a model for that brand
  (`VehicleService.CreateModelAsync` validates via `IsBrandAllowedForUserAsync`; `GetModelsAsync`
  clones on first view).
- ✅ `UserBrandMembership` is purely additive: `AccessScope`/`ManagerBrandScope`/`RequireBrandId()`
  and every controller that reads `cid`/`bid` need **zero changes** — they only ever see the
  active Filial. Switching (`POST /api/users/me/switch-brand`) re-mints a JWT exactly like login,
  it doesn't mutate a live token's claims.
- ✅ Registration inserts a membership row for the Filial a user gets at signup, alongside the
  existing `CompanyId`/`BrandId` assignment.
- ✅ Manager can grant/revoke membership only within their own company (`POST/DELETE
  /api/brands/{brandId}/members[/{userId}]`); Admin can do any company
  (`POST/DELETE /api/admin/users/{id}/memberships`).
- ❌ No live "you've been moved, refresh your session" push — switching is a deliberate user
  action (or the next login), not a forced kick when membership changes elsewhere. Acceptable for
  v1; revisit if it causes confusion in practice.

**Rejected:** keep vehicle brands per-user and just add membership separately → still leaves the
core mismatch (catalog config should follow the business, not the individual) unresolved; doing
both together was cheap since membership directly determines "whose Filial-config applies right
now."

Affects: [[USER-BRANDS]] · [[VEHICLES]] · [[ARCHITECTURE]] · [[SCHEMA-CONFIG]] · [[DOMAIN]]
