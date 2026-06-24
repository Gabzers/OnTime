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

## Requested improvements (reported June 2026)
- **Status indicator on the Vehicles screen** per model:
  - 🔘 grey = missing configuration (no version/colours/price/fuel)
  - 🔴 red = inactive (`IsActive=false`)
  - 🟢 green = active **and** configured
- **Active/inactive toggle per model** on the Vehicles screen (manager controls which models are sellable). Needs a `PATCH /api/vehicles/models/{id}/active` (mirror the brand one) + `VehicleModel.IsActive` already exists.
- **Define "configured"**: at least one version with ≥1 exterior colour (decide the exact rule and document it here when implemented).
- Seed (or let managers add) versions + colours + year/price for the active models, so proposals can pick a real configuration.

## Filtering by the user's brands
Vehicle lists and the proposal vehicle-picker must respect the user's selected brands — see [[USER-BRANDS]].

## Related
[[SCHEMA-CONFIG]] · [[USER-BRANDS]] · [[FRONTEND-ISSUES]] · [[KNOWN-ISSUES]] · [[OnTimeCRM]]
