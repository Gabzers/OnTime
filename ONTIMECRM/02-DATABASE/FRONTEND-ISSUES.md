# Frontend — Known Issues

Hub: [[OnTimeCRM]] · Roadmap: [[ROADMAP]] (Phase 0). Concrete bugs found by reading source on 2026-05-30.

## 🔴 Functional bugs

### 1. ~~Snooze sends the wrong field name~~ ✅ fixed T6 (2026-06-06)
`SnoozeNotificationRequest.snoozeUntil` renamed to `snoozedUntil`; both callers (`NotificationDropdown`, `NotificationsPage`) updated.

### 2. ~~`NotificationsPage` reads a non-existent field~~ ✅ fixed T8 (2026-06-06)
`NotificationDto.dueAt` → `scheduledFor`; `CreateNotificationRequest.dueAt` → `scheduledFor`; `notes` → `body` (matching backend `Body` field); added missing `doneAt?`. `NotificationsPage` updated to `n.scheduledFor`.

### 3. ~~NotificationStatus labels & colors swapped~~ ✅ fixed T7 (2026-06-06)
Fixed in `I18nController` (pt-PT + en-US), `i18nFallback.ts`, `EnumTag.NotificationStatusTag` colors, `NotificationDropdown.statusColor`.

### 4. ~~Theme flash (FOUC) on first load in dark mode~~ ✅ fixed T10 (2026-06-06)
Switching screens while in dark mode flashes white, then settles to dark on first paint of a
screen. The theme is applied after React hydrates instead of before first paint.
**Fix:** apply the persisted `themeStore.isDark` synchronously before render — set the Ant
algorithm + a `dark` class on `<html>` from an inline script / pre-hydration read of
`localStorage('ontimecrm-theme')`, so the initial paint is already dark.

### 5. ~~Raw i18n keys rendered (missing keys)~~ ✅ fixed T12 (2026-06-06)
- Proposal detail shows a button literally labelled `ACTION.PROPOSAL.LOST`.
- Convert-to-sale shows `MSG.CONVERT.SOLD_AT_HINT`.
The component uses keys that don't exist in `/api/i18n` (the hint key is defined as
`HINT.PROPOSAL.SOLD_AT`, not `MSG.CONVERT.SOLD_AT_HINT`). **Fix:** add the missing keys to PtPT/EnUS
+ fallback, or point the component at the existing key. See [[I18N]].

### 6. ~~Back from proposal detail returns to the wrong tab~~ ✅ fixed T13 (2026-06-06)
Opening a proposal detail then "back" lands on the **"Informação"** tab instead of
**"Propostas (1)"**. **Fix:** preserve the originating tab (route state / query param) so back
returns to the proposals tab on the client detail page.

### 7. Encoding mojibake — `ç`/`ã`/`—` render as `Ã¡` / `â€"`
e.g. "Configuração" → "ConfiguraÃ§Ã£o", "Hora da notificação", em-dash → "â€"". The **source is
correct UTF-8** (verified) — this is a runtime/compile encoding issue. Most likely the i18n
`.cs` source compiled as Windows-1252 (UTF-8 **without BOM** on a Windows build), so the string
literals are already corrupt in the assembly. **Fix:** save source files with non-ASCII as UTF-8
**with BOM** (or enforce via `.editorconfig` `charset = utf-8-bom`); verify the `/api/i18n`
response is `application/json; charset=utf-8`. See [[I18N]].

### 8. Vehicles screen — status + active toggle (enhancement)
Needs a status dot per model (grey=unconfigured, red=inactive, green=active+configured) and an
active/inactive toggle. Full spec in [[VEHICLES]].

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
