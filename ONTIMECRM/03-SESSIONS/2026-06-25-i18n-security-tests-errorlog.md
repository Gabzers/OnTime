# i18n overhaul, security audit, test-quality audit, error logging — 2026-06-25

Session log (snapshot, not updated later). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** work through a long backlog one item at a time: full dark/light-mode audit, a
systematic Chrome sweep of every screen, a test-quality audit ("do the tests actually catch
bugs, or just pass?"), and a persisted error-log table — plus, along the way, finish real
English (en-US) UI support and fix a desktop sidebar collapse animation through several
iterations.

**Done:**
- **i18n / English support is now real**, not decorative. ~130 hardcoded Portuguese strings
  moved to `t['...']` across 15+ pages/components. Root cause of "nothing actually translates":
  `App.tsx`'s antd `ConfigProvider` hardcoded `locale={ptPT}` regardless of the selected
  language — fixed to switch with `i18nStore.locale`. New `src/utils/format.ts` (`tFormat`) for
  placeholders in translated strings. Found two keys (`LABEL.NOTIF_SETTINGS.DAILY_DIGEST_TIME`/
  `DIGEST_ENABLED`) that existed only in the frontend's pt-PT-only fallback file, never in the
  backend dictionary — added to both locales. See [[I18N]].
- **Cross-tenant security hole fixed**: `/api/admin/*` only required `ManagerOnly` (role 1 or 2)
  instead of true platform-Admin — any paying customer's Manager could list/disable/edit every
  OTHER company. New `AdminOnly` policy (role 2 only); `/admin` removed from the per-company
  configurable permission set (`PermissionService.AllRoutes`) with a startup DB cleanup for
  pre-existing rows; `AdminFlowTests.cs` added (zero tests existed for this controller before).
  See [[SECURITY]].
- **Error log table**: `ErrorLog` entity + `ErrorHandlingMiddleware` persists every error
  response (4xx ApiException, 409 DB conflict, 500 unhandled w/ stack trace) to `error_logs`.
  `GET /api/admin/error-logs` (AdminOnly, paginated) to read them. Logging failures are
  swallowed so they never mask the real error response; change tracker cleared first so a
  failed save doesn't get re-attempted alongside the log row.
- **Test-quality audit**: found the Friends email/name-swap bug was hidden because tests
  asserted only HTTP status codes, never response body content (added body assertions). Audited
  every "cross-tenant" suspect an automated sweep flagged (Goals, Notifications, Stages,
  NotificationPreferences, Subscription) by reading the actual service code — all were already
  correctly protected (ownership checks like `goal.UserId != userId`), just never asserted by a
  test. Added the missing regression tests rather than re-fixing already-correct code.
- **Dark/light mode audit**: `RegisterPage.tsx` had no dark-mode support at all (no toggle, fixed
  light background); both auth pages' error boxes used hardcoded `bg-red-50` regardless of
  theme; 7 hardcoded Tailwind gray text classes in `ClientDetailPage`/`SaleDetailPage` replaced
  with theme tokens.
- **Goals date-window bug** (found mid-sweep, see [[GOALS-PERMISSIONS]]): a Monthly goal's
  `StartDate` defaulted to "today" in the UI instead of the start of the period, so sales made
  earlier that month were silently excluded from its own progress count. The backend logic was
  always correct (and tested with a manually-constructed start-of-month date) — only the
  frontend's default was wrong. Removed the separate manual `EndDate` field from the UI entirely.
- **Supabase query-performance audit**: drift detector extended to also diff **indexes**
  (`pg_indexes`), not just tables/columns — a `HasIndex(...)` added to an already-deployed
  entity previously never reached an existing DB at all. Fixed `Sale`'s date filter
  (`SoldAt.Year`/`.Month`, can't use an index) → range comparison + new index on `SoldAt`.
  Two lower-priority findings left open: Dashboard's 4 separate round-trips, `SearchUsersAsync`'s
  double query.
- **Desktop sidebar collapse**, several iterations (see "Didn't work" below for the failed
  attempts) — final state: icon position is fixed via `padding-left: 16px` forced regardless of
  antd's own collapsed-mode layout (which otherwise jumps it once, after the rail finishes
  animating); antd's `inlineCollapsed` is never toggled at all anymore, only the label's own
  opacity fades; hover tooltips on collapsed items work again.
- Full Chrome sweep of every screen not previously checked (Proposals, Sale/Proposal detail,
  Stages, Brands, Admin, Access Control, Profile, Notifications, New Client + Trade-In) — found
  and fixed `ProfilePage.tsx`'s "Aparência"/"Idioma"/"Marcas que Vendo" still hardcoded (missed
  by the first i18n sweep).

**Findings / gotchas:**
- A memory/hunch isn't a fact — an agent-produced "15 critical findings" test-quality report
  needed direct source-reading to confirm; most turned out already-correct-but-untested rather
  than live bugs. Verify before acting on an automated audit's claims, especially severity ones.
- `EnsureCreated` truly never alters existing tables — not just missing columns, but missing
  **indexes** too. Any future `HasIndex(...)` addition needs the same drift-check treatment or it
  silently never applies to an already-running DB (local or prod).
- antd's `Tooltip`-on-collapsed-item feature reuses the exact same `label` ReactNode for both the
  rendered menu item and the tooltip's title — styling that node with `opacity:0` for a fade
  effect makes the tooltip pop up empty too. Fade via a scoped CSS class on the *container*
  instead of the label node itself.

**Didn't work / dead ends:**
- Overriding `.ant-layout-sider`'s width transition with `!important` to get a smoother
  animation curve fought antd's own JS-driven collapse bookkeeping (it listens for
  `transitionend`) and froze the rail at the old width for several seconds before randomly
  resyncing — worse than the default. Reverted.
- Forcing the Sider's width via a higher-specificity CSS selector keyed off the
  `ant-layout-sider-collapsed` marker class still lost to antd's own cssinjs-generated class
  (verified: inline style + marker class said 64px, the hashed cssinjs class still said 232px
  `!important`) — a real antd staleness quirk, not fixable by winning the specificity war.
- Replacing antd's `<Layout.Sider>` with a plain `<div>` (to dodge the above) broke the page
  layout entirely — `Layout` only switches to a row (sidebar-beside-content) flex direction when
  it recognizes a real `Sider` child; a plain div made it stack everything vertically instead.
  Reverted to `<Sider>`, fixed the actual icon-jump complaint differently (fixed padding +
  never toggling `inlineCollapsed`) instead of fighting the width mechanism at all.
- Delaying `inlineCollapsed` by a `setTimeout` (so the icon reposition only happened once the
  rail finished shrinking) worked technically but the icon still moved once, at the end — the
  user wanted it to never move at all, which only the fixed-padding approach achieves.

**Next:**
- Dashboard's 4-round-trip aggregation and `SearchUsersAsync`'s double query (both in
  [[KNOWN-ISSUES]]) if Supabase performance becomes a real concern.
- Most `ManagerOnly` controllers (Brands, Users, Vehicles, Permissions) still have no test
  confirming a Salesperson gets 403 — policy is enforced, just not regression-tested.
- The i18n DB-vs-hardcoded architecture decision (Phase 3, [[ROADMAP]]) is still open — now a
  pure architecture question, not a translation-completeness problem.

**Links:** [[ROADMAP]] Phase 1 (Goals/Friends/Permissions audit, now done) · [[KNOWN-ISSUES]] ·
[[SECURITY]] · [[I18N]] · [[GOALS-PERMISSIONS]] · [[FRIENDS]]
