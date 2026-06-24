# Feature: User Brands (vehicle filtering)

Hub: [[OnTimeCRM]] · Status: **planned** · Schema: [[SCHEMA-CONFIG]] · Vehicles: [[VEHICLES]].

## Goal
On the profile, the user picks the vehicle **brands they sell** (e.g. XPENG). **Multiple allowed.**
Everywhere a vehicle is shown or chosen, only models of the user's brands appear — no noise from
brands they don't represent.

> Note: this is the **VehicleBrand** catalogue (XPENG/Voyah/Dongfeng), distinct from the
> company **Brand**/stand the user belongs to. Don't conflate them.

## Data model (proposed)
Many-to-many `User ↔ VehicleBrand`:
```
user_vehicle_brands (user_id, vehicle_brand_id)   PK (user_id, vehicle_brand_id)
```
- New entity `UserVehicleBrand` + EF config; add the join to [[SCHEMA-CONFIG]] when built.
- Empty selection = show all brands (sensible default so nothing breaks for existing users).

## API (proposed)
```
GET /api/users/me/vehicle-brands         → list selected brand ids
PUT /api/users/me/vehicle-brands         → body: { brandIds: Guid[] }
```
And filtering applied server-side:
- `GET /api/vehicles/models` → accept the caller's brand set; default to the user's selected brands when no explicit `brandId` filter is given.
- Anywhere the proposal vehicle-picker loads models.

## Frontend
- Profile page: multi-select of brands (from `GET /api/vehicles/brands`), saved via the PUT above.
- Vehicle list + proposal vehicle-picker: default-filter to the user's brands; allow "show all" override if useful.

## Decisions to confirm (before building)
- [ ] Filtering enforced **server-side** (recommended — consistent everywhere) vs frontend-only.
- [ ] Empty selection = "all brands" (recommended) vs "none".
- [ ] Does a Manager set this per salesperson, or each user sets their own? (default: each user, like other profile prefs)

## Related
[[VEHICLES]] · [[SCHEMA-CONFIG]] · [[DOMAIN]] · [[ROADMAP]] · [[OnTimeCRM]]
