# Reported bug batch + vehicles/brands — 2026-06-06

Session log (snapshot). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** capture a user-reported bug list + two feature requests into the docs; apply the XPENG seed.

## Done (code)
- `Program.cs SeedVehicleBrandsAsync` → made **per-brand idempotent** and added **XPENG** (P7, P7+, P5, MONA M03, G3i, G6, G7, G9, X9). Idempotent so it also lands on an existing local DB.

## Done (docs)
- New: [[VEHICLES]] (catalogue structure, seed table, status-dot + active-toggle request, missing version/colour data) and [[USER-BRANDS]] (per-user multi-brand filter, proposed `user_vehicle_brands` join + API).
- [[FRONTEND-ISSUES]] +5: theme FOUC, raw i18n keys, back-tab nav, encoding mojibake, vehicles status UI.
- [[KNOWN-ISSUES]] +3 backend: proposal-without-vehicle, no version/colour seed, missing i18n keys.
- [[I18N]] expanded: encoding cause, missing keys, pt↔en switching gaps.
- [[SCHEMA-CONFIG]] corrected: `vehicle_model_versions` has `external_colors`/`internal_colors`; added planned `user_vehicle_brands`.
- [[AI-TASK-PLAN]] +T10–T15 (bugs) + F1 (user brands).

## Findings
- Encoding: **source is valid UTF-8** (verified) → runtime/compile issue, almost certainly UTF-8-without-BOM compiled on Windows. Diagnostic: check `/api/i18n` response charset + file BOM.
- Versions/colours empty is **expected** — seed creates brands+models only.
- Proposal-without-vehicle is a real backend validation gap (convert then 422 `SALE_MISSING_VEHICLE`).
- XPENG lineup confirmed via web (2026 global range).

## Next / open decisions
- [[USER-BRANDS]]: confirm server-side filtering + empty=all + who sets it.
- Decide whether to seed versions/colours or build model-config UI.
- These slot after the access-scope helper + trial fix in [[AI-TASK-PLAN]].

## Links
[[VEHICLES]] · [[USER-BRANDS]] · [[FRONTEND-ISSUES]] · [[KNOWN-ISSUES]] · [[I18N]] · [[AI-TASK-PLAN]]
