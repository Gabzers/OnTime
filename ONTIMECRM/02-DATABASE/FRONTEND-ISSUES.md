# Frontend — Known Issues

Hub: [[OnTimeCRM]] · Roadmap: [[ROADMAP]] (Phase 0). Concrete bugs found by reading source on 2026-05-30.

## 🔴 Functional bugs

### 1. ~~Snooze sends the wrong field name~~ ✅ fixed T6 (2026-06-06)
`SnoozeNotificationRequest.snoozeUntil` renamed to `snoozedUntil`; both callers (`NotificationDropdown`, `NotificationsPage`) updated.

### 2. ~~`NotificationsPage` reads a non-existent field~~ ✅ fixed T8 (2026-06-06)
`NotificationDto.dueAt` → `scheduledFor`; `CreateNotificationRequest.dueAt` → `scheduledFor`; `notes` → `body` (matching backend `Body` field); added missing `doneAt?`. `NotificationsPage` updated to `n.scheduledFor`.

### 3. ~~NotificationStatus labels & colors swapped~~ ✅ fixed T7 (2026-06-06)
Fixed in `I18nController` (pt-PT + en-US), `i18nFallback.ts`, `EnumTag.NotificationStatusTag` colors, `NotificationDropdown.statusColor`.

### 4. ~~Theme flash (FOUC) on first load in dark mode~~ ✅ fixed T10 (2026-06-24)
Switching screens while in dark mode used to flash white, then settle to dark on first paint of a
screen. The theme was applied after React hydrated instead of before first paint.
**Fix:** read the persisted `themeStore.isDark` synchronously before render, sync the `dark` class on
`<html>`, and keep that state aligned during rehydrate so the initial paint stays dark.

### 5. ~~Raw i18n keys rendered (missing keys)~~ ✅ fixed T12 (2026-06-24)
- Proposal detail showed a button literally labelled `ACTION.PROPOSAL.LOST`.
- Convert-to-sale showed `MSG.CONVERT.SOLD_AT_HINT`.
The backend i18n payload is still partial in some environments, so the frontend now merges it over
`PT_PT_FALLBACK` before rendering. This prevents raw keys from leaking even when `/api/i18n`
doesn't include every string. See [[I18N]].

### 6. ~~Back from proposal detail returns to the wrong tab~~ ✅ fixed T13 (2026-06-24)
Opening a proposal detail then "back" used to land on the **"Informação"** tab instead of
**"Propostas (1)"**. **Fix:** keep the client tab in controlled state and restore it from the
navigation state when returning from proposal detail.

### 7. ~~Encoding mojibake — `ç`/`ã`/`—` render as `Ã¡` / `â€"`~~ fixed 2026-06-24 (found the real instance)
First pass: checked `curl http://localhost:8080/api/i18n?locale=pt-PT` against the running
container — correctly encoded UTF-8, no BOM issue, not the `/api/i18n` payload at all.
Second pass (live browser testing on `/notification-settings`): the user actually saw `â€"` in
the rendered "Hora da notificação" column. Root cause found by grepping for `â€` across the repo —
**`NotificationSettingsPage.tsx` had literal corrupted bytes committed to source**: the fallback
string was `'â€”'` (mojibake) instead of `'—'` (proper em dash), plus `â”€` mojibake in several
section-divider comments. This is a genuinely corrupted source file, unrelated to BOM/Roslyn/i18n —
fixed by rewriting the file as proper UTF-8. No other file in the repo had this pattern (verified
via repo-wide grep for `â€|â”€`).

### 8. ~~Vehicles screen — status + active toggle~~ ✅ fixed T15 (2026-06-06)
Added a status dot per model (grey=unconfigured, red=inactive, green=active+configured) and an
active/inactive toggle. Full spec in [[VEHICLES]].

### 9. New-client vehicle picker behavior ✅ fixed 2026-06-24
`NewClientPage` now locks the brand selector when the user has exactly one selected vehicle brand,
allows choosing among selected brands when there are multiple, and only shows active models.

## 🟠 i18n rule violations (hardcoded pt-PT) ✅ fixed T9 (2026-06-06)
- `NotificationDropdown.tsx`: `title="Adiar 1h"` → `t['ACTION.NOTIFICATION.SNOOZE_1H']`; `"Ver todas"` → `t['ACTION.VIEW_ALL']`
- `api/axios.ts`: 403 handler now reads `MSG.ERROR.FORBIDDEN` from i18nStore

## 🟡 Typing / minor — partially fixed T9
- ~~`App.tsx`: `(user as any)?.brandColor`~~ — fixed, `brandColor` was already typed, dropped `as any`
- ~~`usePermissions.ts`: `(user as any)?.role`~~ — fixed
- ~~`MenuPermissionDto`/`UpdateMenuPermissionRequest` only exposed 2 flags~~ — fixed, now 4 flags; `AccessControlPage` expanded to 4 columns
- i18n has `ENUM.TRANSMISSION.*` and `ENUM.VEHICLE_SEGMENT.*` keys, but `VehicleModel` has no such fields. Dead keys — low priority.
- `NotificationsPage` only shows `/today`; paginated history (`getPaged`) unused.

## Audit checklist (Phase 0)
- [ ] Cross-check every `api/*.ts` request/response shape against backend DTOs (field names, casing)
- [ ] Cross-check every `ENUM.*` i18n label against [[DOMAIN]] canonical values
- [ ] **Cross-check every i18n key used in components against the keys defined in `/api/i18n`** (items 5 shows several are missing) — and that **pt↔en switching** covers all of them (reported: many strings don't switch)
- [ ] Grep components for hardcoded Portuguese string literals
- [ ] Verify mobile vs desktop rendering on each page (`ResourcePage`)

## Related
[[NOTIFICATIONS]] · [[I18N]] · [[VEHICLES]] · [[DOMAIN]] · [[CONVENTIONS]] · [[ROADMAP]] · [[OnTimeCRM]]
