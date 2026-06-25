# Sidebar / mobile drawer redesign — 2026-06-24

Session log (snapshot, not updated later). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** the user found the sidebar nav "ugly" on both desktop and mobile and asked for a
complete redesign.

**Done:**
- Rewrote `Sidebar.tsx`: flat 12-item icon list → 4 labelled groups (Geral / Vendas / Operação /
  Configuração) via AntD `Menu` `type: 'group'` items; added a branded header (colour mark "OT" +
  wordmark + the user's `brandName`, accent colour from `user.brandColor`); nicer user footer
  (avatar + name + role label instead of just an avatar + name).
- Component now takes `variant: 'sider' | 'drawer'`:
  - `sider` (desktop, inside `AppLayout`'s `Layout.Sider`, 232px): click-to-open dropdown for
    profile/logout, matching prior behaviour but restyled.
  - `drawer` (mobile, inside `AppLayout`'s `Drawer`): explicit close (✕) button in the header,
    visible logout icon-button in the footer instead of a dropdown (fewer taps, more discoverable
    on touch), and calls `onNavigate` on every menu click so **the drawer now closes itself when
    you pick a destination** — previously it stayed open over the new page.
- Fixed `isManager` only checking `role === 1` (Manager), which hid the manager-only items
  (Marcas/Admin/Controlo de Acesso) from Admin (`role === 2`) accounts even though Admin is
  supposed to have at least as much access as Manager everywhere else in the app. Now
  `role === 1 || role === 2`.
- New i18n keys `NAV.GROUP.OVERVIEW/SALES/WORKSPACE/SETTINGS` (pt-PT + en-US + frontend fallback).

**Findings / gotchas:**
- The remote Chrome automation environment used for visual verification got stuck reporting a
  fixed ~210×467 / ~412×915 viewport for the rest of the session regardless of
  `resize_window` calls (even after opening a brand-new tab) — could not get a genuine desktop-
  width screenshot to confirm the `Layout.Sider` rendering visually. Verified the mobile
  `Drawer` variant visually (grouping, branding, active-item highlight, auto-close-on-navigate all
  confirmed working); the desktop `Sider` variant is **code-reviewed but not visually confirmed**
  this session — worth a quick look next time a real desktop-width session is available.

**Didn't work / dead ends:**
- `resize_window` + new tab did not recover a wider viewport once the session's virtual display
  appeared to lock at mobile width — don't keep retrying it within the same session past 2-3
  attempts, it's a session-level environment issue, not something fixable by changing tool args.

**Next:**
- Confirm the desktop `Sider` visually next session (group headers, active-item pill, brand-colour
  accent, footer dropdown) and adjust spacing if anything looks off at 232px width.

**Links:** [[STATUS]] · [[CONVENTIONS]] (Sidebar component pattern note)
