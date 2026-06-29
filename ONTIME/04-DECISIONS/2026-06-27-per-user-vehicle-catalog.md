# Vehicle catalog is per-user, not global-with-filter

Date: 2026-06-27 · Status: **implemented 2026-06-27** · [[OnTime]]

**Rule:** every user owns their own copy of the vehicle models/versions they sell
(`UserVehicleModel`/`UserVehicleVersion`). The global `VehicleBrand` list is only the shared
"shelf" to pick from — selecting a brand clones its models into the user's own catalog.

**Why:** each dealership sells a different, mostly non-overlapping lineup. A shared mutable
`VehicleModel`/`VehicleModelVersion` table meant one user editing or deleting a model affected
every other tenant on the platform — fine for a filter, wrong for ownership.

**Implies (do / don't):**
- ✅ `ProposalVehicle.ModelId`/`VersionId` and `Sale.ModelId` FK to `UserVehicleModel`/
  `UserVehicleVersion`, not the global tables.
- ✅ Selecting a brand (`PUT /api/users/me/vehicle-brands` or creating a model for an unselected
  brand) clones only that brand's global models — not the whole shelf — into the user's catalog.
  Re-selecting a previously-selected brand reactivates the same rows, never re-clones.
- ✅ Unselecting a brand hides its models everywhere (`GetModelsAsync` only returns models whose
  brand currently has a matching `UserVehicleBrand` row) — **never deletes**. Reversible.
- ✅ Deleting one specific model/version (`DELETE /api/vehicles/models/{id}`) **is real delete**,
  blocked with `VEHICLE_MODEL_IN_USE` (422) if referenced by the user's own proposal/sale.
- ✅ `VehiclesController` model/version write endpoints dropped `ManagerOnly` — every role manages
  their own catalog now. Brand CRUD (the shared shelf) stays `ManagerOnly`.
- ✅ All model/version repository/service methods take `userId` and scope every lookup by it —
  one user can never read/edit/delete another's catalog row by guessing its id (404, not 403, to
  avoid leaking existence).
- ❌ No data migration was written — this shipped before any production demo data existed, so
  existing-user backfill was explicitly out of scope (confirmed with the user).

**Rejected:** keep the global+filter model, just block cross-user *writes* → still leaves
deletes/edits ambiguous in intent (would a user expect to edit a row everyone shares?) and doesn't
solve "I don't want to see/manage the other 40 brands I don't sell" as cleanly as true ownership.

Affects: [[USER-BRANDS]] · [[VEHICLES]] · [[DOMAIN]] · [[ARCHITECTURE]]
