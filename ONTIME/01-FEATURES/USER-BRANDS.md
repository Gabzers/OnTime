# Feature: Filial vehicle brands + multi-Filial membership

Hub: [[OnTime]] · Status: **redesigned 2026-06-27 (twice)** · Schema: [[SCHEMA-CONFIG]] · Vehicles: [[VEHICLES]].

## Goal
1. **Which car brands a Filial sells** is configured once per Filial by Manager/Admin
   (`BrandVehicleBrand`) — not picked individually by each salesperson anymore.
2. **The personal vehicle catalog stays per-user** (`UserVehicleModel`/`UserVehicleVersion`,
   create/edit/delete) — only the *source list* of which `VehicleBrand`s are allowed changed.
3. **A user can belong to more than one company/Filial** (`UserBrandMembership`) and switch which
   one is "active" — switching changes which Filial's vehicle brands apply to them.

> Note: this is the **VehicleBrand** catalogue (XPENG/Voyah/Dongfeng), distinct from the company
> **Brand**/Filial the user belongs to. Don't conflate them — this doc calls the latter "Filial".

## History (same sprint, two redesigns)
1. **2026-06-06**: global `VehicleModel` catalogue, filtered per-user via `UserVehicleBrand`.
2. **2026-06-27 (morning)**: per-user *ownership* — `UserVehicleModel`/`UserVehicleVersion`, cloned
   from the global catalogue when a user picked a brand. Fixed "editing a model affects every
   tenant", but still had each salesperson individually picking brands.
3. **2026-06-27 (afternoon)**: vehicle-brand *configuration* moved to the Filial (Manager/Admin),
   while keeping per-user catalog ownership — plus multi-Filial membership, designed together
   because membership determines which Filial's config currently applies to a user.

## Data model
- `VehicleBrand` (global) — the shared "shelf". No public endpoint creates models under it
  directly; `ManagerOnly` can still create/delete the brand itself
  (`POST/DELETE /api/vehicles/brands`).
- `BrandVehicleBrand` (`BrandId, VehicleBrandId`) — which car brands a **Filial** sells. Replaces
  the old per-user `UserVehicleBrand` entirely (dropped, no migration needed — no prod data).
  `ManagerOnly`, scoped to the manager's own company (`RequireCompanyId()`).
- `UserVehicleModel`/`UserVehicleVersion` — the personal catalog, FK'd to `UserId`. Unchanged from
  the morning redesign. `ProposalVehicle`/`Sale` FK here.
- `UserBrandMembership` (`UserId, BrandId`) — the Filiais a user is allowed to switch into.
  `User.CompanyId`/`BrandId` (and the JWT's `cid`/`bid` claims) stay the **currently active**
  Filial — membership doesn't change how `AccessScope`/`ManagerBrandScope` work, it's purely the
  set a user can switch between. See [[ARCHITECTURE]].

## Selecting / unselecting a brand for a Filial (`BrandRepository.SetVehicleBrandIdsAsync`)
- **Allow a brand the Filial never had**: the next time each of the Filial's users views their
  models (or creates one), `VehicleRepository` lazily clones that brand's global models/versions
  into *their own* `UserVehicleModel`/`UserVehicleVersion` — once per user, idempotent.
- **Disallow a brand**: just removes the `BrandVehicleBrand` row. Every user of that Filial stops
  seeing/creating models for it (`VehicleRepository.GetModelsAsync` only returns models whose
  brand is in `BrandVehicleBrand` for the calling user's active Filial) — their cloned rows are
  **not deleted**, so re-allowing the brand makes them reappear exactly as they were, no
  duplication. This is independent of a model's own manual `IsActive` toggle (the red/green status
  dot) — that's a separate, narrower concern.

## Manual model creation requires an allowed brand
`POST /api/vehicles/models` checks `VehicleRepository.IsBrandAllowedForUserAsync` (the calling
user's active Filial must have that `VehicleBrand` in `BrandVehicleBrand`) — `403
VEHICLE_BRAND_NOT_ALLOWED` otherwise. No more auto-registering a brand from a manual create; that
privilege now belongs to Manager/Admin only.

## Deleting a model/version
Unchanged from the morning redesign: real delete, blocked with `VEHICLE_MODEL_IN_USE` (422) if
referenced by the user's own proposal/sale.

## Multi-Filial membership API
```
GET  /api/users/me/memberships              → [{ companyId, companyName, brandId, brandName }]
POST /api/users/me/switch-brand             → body { brandId }, must already have membership,
                                                 returns a freshly minted LoginResponseDto (new JWT)
POST/DELETE /api/admin/users/{id}/memberships          → AdminOnly, any company
POST/DELETE /api/brands/{brandId}/members[/{userId}]   → ManagerOnly, own company only
```
Registration (`RegisterManagerAsync`/`RegisterSalespersonAsync`) inserts a `UserBrandMembership`
row for the Filial the user gets at signup, alongside setting `CompanyId`/`BrandId` as today.

## Vehicle-brands-per-Filial API
```
GET/PUT /api/brands/{id}/vehicle-brands     → { vehicleBrandIds: Guid[] }, ManagerOnly, own company
```
`VehiclesController` model/version write endpoints have **no role restriction** — every role
manages their own personal catalog; brand CRUD (the global shelf) and the per-Filial vehicle-brand
config stay Manager/Admin-only.

## Frontend
- Sidebar header (the Filial name shown under "OnTime") becomes a clickable `Dropdown` only when
  the user has more than one membership — picking another Filial calls `switch-brand` and swaps
  the stored JWT like a fresh login. Single-membership users see unchanged static text.
  `Sidebar.tsx`.
- `ProfilePage.tsx` — "Marcas que Vendo" card removed entirely (no longer a per-user concern).
- `BrandsPage.tsx` — new drawer per Filial: vehicle-brands multi-select (same UX the old Profile
  card had) + a basic membership-grant control. `AdminPage.tsx`'s `UsersSubTable` gained an
  "Atribuir Filial" action (company+brand picker, any company).
- `VehiclesPage.tsx`/`NewClientPage.tsx` — brand-filter/lock logic now reads
  `brandsApi.getVehicleBrands(user.brandId)` instead of the removed per-user endpoint.

## Tests
- `BrandVehicleBrandsFlowTests.cs` — Filial-level config CRUD + cross-company 403, clone-on-first-
  view, hide-on-disallow (row still in DB), reactivate-without-duplicating, manual create blocked
  for a disallowed brand, tenant isolation across two Filiais, delete blocked when in use.
- `MembershipFlowTests.cs` — registration grants membership, switch without membership is 403,
  Admin grants cross-company + user can then switch, Manager can only grant within their own
  company, revoke removes the ability to switch.

## Decisions — confirmed 2026-06-27
- [x] Vehicle brands are configured **per-Filial**, not per-salesperson.
- [x] Personal catalog ownership (create/edit/delete models/versions) **stays per-user**.
- [x] Multi-Filial membership is **additive** — doesn't change JWT claim shape or `AccessScope`.
- [x] Switching Filial = re-login under the hood (fresh JWT), not a live claim mutation.
- [x] No data migration needed (shipped before any production demo data existed).

## Related
[[VEHICLES]] · [[SCHEMA-CONFIG]] · [[DOMAIN]] · [[ARCHITECTURE]] · [[ROADMAP]] · [[OnTime]]
