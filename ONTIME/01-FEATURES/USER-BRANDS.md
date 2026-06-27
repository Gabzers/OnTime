# Feature: User Brands (vehicle filtering)

Hub: [[OnTime]] · Status: **✅ done (2026-06-06)** · Schema: [[SCHEMA-CONFIG]] · Vehicles: [[VEHICLES]].

## Goal
On the profile, the user picks the vehicle **brands they sell** (e.g. XPENG). **Multiple allowed.**
Everywhere a vehicle is shown or chosen, only models of the user's brands appear — no noise from
brands they don't represent.

> Note: this is the **VehicleBrand** catalogue (XPENG/Voyah/Dongfeng), distinct from the
> company **Brand**/stand the user belongs to. Don't conflate them.

## Data model (implemented)
Many-to-many `User ↔ VehicleBrand` via `UserVehicleBrand : BaseEntity`
(`OnTime_Backend/src/OnTime.Domain/Entities/UserVehicleBrand.cs`):
```
user_vehicle_brands (id PK, user_id, vehicle_brand_id, ...BaseEntity fields)
unique index (user_id, vehicle_brand_id)
```
- Own `Guid Id` (not composite PK) — matches the `MenuItemPermission` pattern in this codebase.
- `User.SelectedVehicleBrands` nav collection; cascade delete both sides.
- Empty selection = show all brands (no behavior change for existing users).
- No EF migration needed — `EnsureCreated` auto-detects the new table (see `DatabaseInitializer.IsSchemaCurrentAsync`).

## API (implemented)
```
GET /api/users/me/vehicle-brands         → { brandIds: Guid[] }
PUT /api/users/me/vehicle-brands         → body: { brandIds: Guid[] }   (replaces the full set)
```
Filtering applied server-side in `VehicleRepository.GetModelsAsync(p, userId, ct)`:
- If the request has an explicit `?brandId=`, it wins (manual override).
- Otherwise, if the calling user has any selected brands, results are filtered to those.
- Otherwise (no selection), no filter — all brands shown.
- `IVehicleService.GetModelsAsync` and `VehiclesController.GetModels` both take `userId` now.

## Frontend (implemented)
- `ProfilePage`: "Marcas que Vendo" card — `Select mode="multiple"` populated from `vehiclesApi.getBrands()`,
  value/save via `usersApi.getMyVehicleBrands()` / `usersApi.updateMyVehicleBrands()`.
- `VehiclesPage` model list already calls `GET /api/vehicles/models` without a brand filter by default,
  so it's automatically scoped server-side — no frontend change needed there.
- `NewClientPage`: if the user has exactly one selected vehicle brand, the vehicle picker locks that brand and pre-fills it; if there are multiple selected brands, the picker lets the user choose between them.
- `NewClientPage`: the proposal vehicle picker in the new-client flow only shows active models.
- `VehicleProposalTable` (proposal vehicle-picker) always passes an explicit `brandId` once the user picks
  a brand for that row — explicit `brandId` deliberately overrides the user's selection (see API rule above),
  so the **model** dropdown there is unaffected, by design.
- **Gap**: the **brand** dropdown inside `VehicleProposalTable` calls `vehiclesApi.getBrands()`, which is
  not filtered to the user's selected brands — a salesperson still sees every brand in that picker, not just
  the ones they sell. Low priority since picking the "wrong" brand there is harmless (just noise), but worth
  fixing if this becomes confusing in practice.

## Tests
`UserVehicleBrandsFlowTests.cs` (6 tests): empty default, persist selection, clear selection,
no-selection-shows-all, selection-filters, explicit `brandId` overrides selection.

## Decisions — confirmed 2026-06-06
- [x] Filtering enforced **server-side**.
- [x] Empty selection = **all brands** (no behavior change for existing users).
- [x] **Each user sets their own** brands in `/profile` (no manager-assignment screen).

## Related
[[VEHICLES]] · [[SCHEMA-CONFIG]] · [[DOMAIN]] · [[ROADMAP]] · [[OnTime]]
