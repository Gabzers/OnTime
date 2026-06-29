# Feature: Vehicle Catalogue

Hub: [[OnTime]] · Schema: [[SCHEMA-CONFIG]] · API: [[API-REFERENCE]] · Per-user model: [[USER-BRANDS]].

Three layers, redesigned twice on 2026-06-27 (see [[USER-BRANDS]] for the full history):
- **Global "shelf"**: `VehicleBrand` only (XPENG/Voyah/Dongfeng/...) — shared, the list everyone
  picks from. No public endpoint creates models under it; populated out-of-band (seed today, an
  admin screen possibly later).
- **Per-Filial config**: `BrandVehicleBrand` — which car brands a company's Filial sells,
  configured by Manager/Admin. Drives which `VehicleBrand`s each of that Filial's users can clone.
- **Per-user catalog**: `UserVehicleModel` → `UserVehicleVersion`, owned by each user, cloned from
  the global brand's models the first time their Filial allows that brand and they view/create a
  model (see [[USER-BRANDS]] for the full clone/hide/reactivate/delete rules). Proposals attach a
  vehicle from the **calling user's own** catalog (or free-text if not catalogued).

## Structure
```
VehicleBrand (global)
UserVehicleModel → UserVehicleVersion   (per user, FK UserId)
                     ├─ ExternalColors[] (serialized via ColorArrayHelper)
                     └─ InternalColors[]
```
- `UserVehicleModel`: Name, Version, Year, FuelType, BasePrice, ImageUrl, IsActive, VehicleBrandId
  (FK to the global brand, for grouping/display only).
- `UserVehicleVersion`: Name + ExternalColors[] + InternalColors[] (colors live on the **version**,
  stored as a serialized array string). A proposal's `ProposalVehicle` also has single
  `ExternalColor`/`InternalColor` (the chosen colour for that deal).

## Seed (Program.cs `SeedVehicleBrandsAsync`, idempotent per brand)
Still seeds the **global** brands + models (the shelf), unchanged:

| Brand | Models |
|-------|--------|
| Dongfeng | AX7, AX4, AX3, T5 EVO, ix5, 580 Pro, AEOLUS E70, Forthing 580 Pro, Forthing T5 EVO, Mengshi 917, Mengshi 919 |
| Voyah | Free, Dream, Passion, Range-E |
| **XPENG** | P7, P7+, P5, MONA M03, G3i, G6, G7, G9, X9 |

> Seed creates **global brands + models only — no versions, no colors, no year/price/fuel**.
> A user only gets these once they select that brand (clones them into their own catalog) — and
> still needs to add versions/colors themselves before a model is "ready" to sell (see below).

## Status indicator + active toggle ✅ done (T15, 2026-06-06; ownership moved to per-user 2026-06-27)
- **Status dot** per model on the Vehicles screen, driven by `getVehicleStatusColor()` (`OnTime_Frontend/src/utils/vehicleStatus.ts`):
  - 🔘 grey = not configured (`IsConfigured=false`)
  - 🔴 red = configured but inactive (`IsActive=false`)
  - 🟢 green = active **and** configured
- **"Configured" rule** (implemented in `VehicleRepository.GetModelsAsync`): at least one
  `UserVehicleVersion` with a non-empty `ExternalColors` array.
- **Active/inactive toggle**: `Switch` in the Vehicles table — **now available to every role**,
  not just managers, since the model belongs to the calling user → `PATCH /api/vehicles/models/{id}/active`
  (`SetVehicleModelActiveRequest { IsActive }`). `UserVehicleModel.IsActive` comes from `BaseEntity`
  (default `true`) — purely a manual flag, independent of brand-selection visibility.
- `VehicleModelListDto` / `VehicleModelDto` both expose `IsActive` and (list only) `IsConfigured`.
- `GET /api/vehicles/models/{id}` hydrates existing vehicle rows before editing — ownership-checked
  (404 if the model isn't the calling user's).
- `NewClientPage` uses the user's selected vehicle brands to lock the brand selector when only one
  brand is available, and only shows active models in that flow.

## "Configurado" filter ✅ done (2026-06-29)
- `VehicleSearchParams.Configured` (`bool?`) — `true` returns only `IsConfigured=true` models,
  `false` only `IsConfigured=false`, omitted/null returns both. Filters server-side inside
  `VehicleRepository.GetModelsAsync`, same query as the brand/search filters.
- `VehiclesPage`'s filter bar gained a second `Select` ("Configurados" / "Não Configurados" /
  "Todas") next to "Filtrar por marca" — **defaults to `true`** on page load, so the management
  list itself opens already scoped to sellable models, not just the proposal picker (see
  [[CONVENTIONS]] "Configured-only vehicle selection").
- Regression test: `VehicleFlowTests.GetModels_FilterByConfigured_ReturnsOnlyMatchingStatus`.

## "Not an automotive account" toggle ✅ done (2026-06-29)
- `Brand.IsAutomotive` (default `true`) — Filial-level, not per-user. Manager/Admin toggle it via
  `BrandsPage`'s edit form ("Vende veículos?" Switch + hint text).
- `LoginResponseDto`/the switch-brand response both carry `isAutomotive` so the frontend has it
  immediately without an extra request — same pattern as `brandColor`/`brandName`.
- When off: `Sidebar` hides the "Veículos" nav item; `NewClientPage`, `ClientDetailPage`'s edit
  modal, and `ProposalDetailPage`'s edit modal all hide their `VehicleProposalTable` section
  entirely. The read-only vehicle list on `ProposalDetailPage`'s detail view (not the edit form)
  is left alone — it only renders when a proposal already has vehicles attached.
- Backend also relaxes `PROPOSAL_MISSING_VEHICLE`: `ClientService.CreateAsync` and
  `ProposalService.CreateForClientAsync` check `IUserRepository.IsAutomotiveAsync(userId)` and skip
  the "≥1 vehicle required" rule when false — otherwise a non-automotive tenant could never create
  a proposal once the picker is hidden from the UI.
- **Caveat:** takes effect on next login or Filial switch, not live-pushed to an already-open
  session — same as the multi-Filial membership switcher (see [[USER-BRANDS]]).
- Tests: `AutomotiveAccountFlowTests.cs` (5 tests) — default true, toggling reflects on next login,
  non-automotive can create with zero vehicles, automotive still requires one, standalone proposal
  creation (not just the inline client+proposal flow) also respects the toggle.

## Still open
- Versions + colours + year/price for cloned models still need to be added by each user before a
  model is "ready" to sell — seeding only ever covered the global shelf, not anyone's personal copy.
- Inactive models are still visible in the generic proposal vehicle-picker outside the new-client
  flow; only the new-client flow filters to active models.
- No admin/manager screen exists yet to author the global "shelf" catalog beyond the hardcoded
  seed — today it's effectively frozen unless edited directly in the DB.

## Per-user ownership, per-Filial configuration
Every model/version a user sees, creates, edits, or deletes belongs to them. Which `VehicleBrand`s
they can do that for is configured once per Filial by Manager/Admin — see [[USER-BRANDS]] for the
full clone-on-first-use / hide-on-disallow / delete-blocked-if-in-use rules and the API/test
inventory.

## Related
[[SCHEMA-CONFIG]] · [[USER-BRANDS]] · [[FRONTEND-ISSUES]] · [[KNOWN-ISSUES]] · [[OnTime]]
