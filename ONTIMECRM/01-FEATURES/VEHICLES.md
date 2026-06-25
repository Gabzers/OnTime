# Feature: Vehicle Catalogue

Hub: [[OnTimeCRM]] · Schema: [[SCHEMA-CONFIG]] · API: [[API-REFERENCE]].

Shared, global catalogue (not per-tenant) of brands → models → versions. Proposals attach a
vehicle (a model, or free-text if not catalogued). This dealership sells Chinese EVs.

## Structure
```
VehicleBrand → VehicleModel → VehicleModelVersion
                                 ├─ ExternalColors[] (serialized via ColorArrayHelper)
                                 └─ InternalColors[]
```
- `VehicleModel`: Name, Version, Year, FuelType, BasePrice, ImageUrl, IsActive.
- `VehicleModelVersion`: Name + ExternalColors[] + InternalColors[] (colors live on the **version**, stored as a serialized array string). A proposal's `ProposalVehicle` also has single `ExternalColor`/`InternalColor` (the chosen colour for that deal).

## Seed (Program.cs `SeedVehicleBrandsAsync`, idempotent per brand)
| Brand | Models |
|-------|--------|
| Dongfeng | AX7, AX4, AX3, T5 EVO, ix5, 580 Pro, AEOLUS E70, Forthing 580 Pro, Forthing T5 EVO, Mengshi 917, Mengshi 919 |
| Voyah | Free, Dream, Passion, Range-E |
| **XPENG** | P7, P7+, P5, MONA M03, G3i, G6, G7, G9, X9 |

> Seed creates **brands + models only — no versions, no colors, no year/price/fuel**. That's why
> the versions endpoint and colour pickers come back empty (see [[KNOWN-ISSUES]]). Models need
> configuration before they're "ready" to sell.

## Status indicator + active toggle ✅ done (T15, 2026-06-06)
- **Status dot** per model on the Vehicles screen, driven by `getVehicleStatusColor()` (`OnTimeCRM_Frontend/src/utils/vehicleStatus.ts`):
  - 🔘 grey = not configured (`IsConfigured=false`)
  - 🔴 red = configured but inactive (`IsActive=false`)
  - 🟢 green = active **and** configured
- **"Configured" rule** (implemented in `VehicleRepository.GetModelsAsync`): at least one `VehicleModelVersion` with a non-empty `ExternalColors` array.
- **Active/inactive toggle**: `Switch` in the Vehicles table (managers only) → `PATCH /api/vehicles/models/{id}/active` (`SetVehicleModelActiveRequest { IsActive }`, mirrors `BrandsController.SetActive`). `VehicleModel.IsActive` comes from `BaseEntity` (default `true`) — no migration needed.
- `VehicleModelListDto` / `VehicleModelDto` now both expose `IsActive` and (list only) `IsConfigured`.
- `GET /api/vehicles/models/{id}` is available for the proposal editor to hydrate existing vehicle rows before editing.
- `NewClientPage` uses the user's selected vehicle brands to lock the brand selector when only one brand is available, and only shows active models in that flow.

## Still open
- Seed (or let managers add) versions + colours + year/price for the active models, so proposals can pick a real configuration.
- Inactive models are still visible in the generic proposal vehicle-picker outside the new-client flow; only the new-client flow filters to active models.

## Filtering by the user's brands
Vehicle lists and the proposal vehicle-picker must respect the user's selected brands — see [[USER-BRANDS]].

## Related
[[SCHEMA-CONFIG]] · [[USER-BRANDS]] · [[FRONTEND-ISSUES]] · [[KNOWN-ISSUES]] · [[OnTimeCRM]]
