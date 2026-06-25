# Frontend regression fix — 2026-06-24

Session log (snapshot, not updated later). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** revalidate and re-fix the frontend regressions reported by the user after the earlier Claude pass.

**Done:**
- Updated `src/store/themeStore.ts` to read persisted dark mode before first paint and sync the `dark` class on `<html>`.
- Updated `src/store/i18nStore.ts` to merge API translations over `PT_PT_FALLBACK`, preventing raw i18n keys from rendering.
- Changed `src/pages/ClientDetailPage.tsx` to control the active tab and restore it from navigation state.
- Scoped `src/pages/ProfilePage.tsx` / `src/pages/VehiclesPage.tsx` vehicle-brand queries to the current user and invalidated the vehicle list on save.
- Updated `src/utils/queryKeys.ts` so `myVehicleBrands` is user-scoped.
- Rebuilt the Docker stack with `docker compose up -d --build`.
- Added vault updates in `[[STATUS]]`, `[[KNOWN-ISSUES]]`, and `[[FRONTEND-ISSUES]]`.

**Findings / gotchas:**
- The earlier fix had drifted: the app still rendered raw keys when the backend i18n map was partial.
- The dark-mode flash was caused by the initial paint not reading persisted state synchronously.
- The back-navigation issue was tied to uncontrolled tab state on the client detail page.

**Didn't work / dead ends:**
- Test runner path resolution for `runTests` did not resolve the frontend test file, so validation used `npm run build` instead.

**Next:**
- If the user wants, continue with the remaining backlog item for the vehicle selector/model filtering behavior.

**Links:** [[FRONTEND-ISSUES]] · [[KNOWN-ISSUES]] · [[STATUS]] · [[ROADMAP]]