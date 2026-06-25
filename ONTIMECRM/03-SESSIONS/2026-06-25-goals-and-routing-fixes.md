# Goals correctness + Admin routing + schema-drift detection — 2026-06-25

Session log (snapshot, not updated later). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** user asked why Goals never show on the home screen, whether all metric/period
combinations actually work, why Marcas/Admin/Controlo de Acesso bounce back to the dashboard,
and to see what a sent friend request looks like.

**Done:**
- **Goals progress was hardcoded to "this calendar month"** regardless of the goal's own period —
  `UserGoalService` reused `dashboard.SalesThisMonth` etc. for every goal. Rewrote
  `ComputeCurrentValueAsync` to query Sales/Proposals/Clients directly, scoped to
  `[StartDate, EndDate ?? PeriodEnd)`. Added `GoalPeriod.Annual` (was missing — only
  Daily/Weekly/Monthly existed). Added `UserGoal.ShowOnDashboard` end-to-end + a new "Os Meus
  Objetivos" widget on `DashboardPage.tsx` (goals previously only ever appeared on `/goals`).
- **`ProtectedRoute`'s `managerOnly` guard checked `role !== 1` only** — Admin (role=2) could see
  Marcas/Admin/Controlo de Acesso in the sidebar (fixed in a prior session) but the route itself
  bounced them back to `/dashboard` on click. Fixed to match Sidebar's `role 1 || 2`.
- **Friend request card was unreadable in dark mode** — `FriendsPage`'s pending-request `<List>`
  hardcoded `background: '#fff'`; sender name/email were invisible (light text on white card on
  dark page). Switched to `theme.useToken().colorBgContainer`.
- **Generic schema-drift detection** — `DatabaseInitializer.IsSchemaCurrentAsync` only ever
  checked one hardcoded sentinel column (`notification_preferences.new_client_notification_days_after`).
  Adding `UserGoal.ShowOnDashboard` reproduced exactly the failure this was supposed to catch: a
  500 on insert ("column does not exist") while the drift check said "schema current". Rewrote to
  compare every table *and* column the EF model expects against `information_schema`.
- Mobile header-overlap pattern (title + action link not wrapping) recurred twice more this
  session (Dashboard's new Goals widget, and is the same class of bug fixed last session on the
  Hot Deals card) — same fix applied (custom flex-wrap `<div>` as the whole `title`, no separate
  `extra`).

**Findings / gotchas:**
- **The generic schema-drift fix wiped all local demo data once.** `EnsureCreated` never adds
  missing columns to existing tables, so the DB had been silently drifting from the EF model for
  a long time — the old single-sentinel check just never noticed because that one column
  happened to already exist. The new check is comprehensive and correctly found the real,
  accumulated drift the first time it ran, triggering the documented drop+recreate. This is the
  *intended* behavior of the no-migrations approach for local dev (see [[CONVENTIONS]],
  [[KNOWN-ISSUES]]) — not a bug, but worth calling out loudly since it's surprising in the
  moment. `AdminBootstrap` reseeds a fresh admin + default stages automatically, which is why
  login kept working through it.
- A `DateTimeOffset.UtcNow.Date` expression in a test produced a 500 that only reproduced inside
  the xUnit/TestServer environment, not via `curl` against the live container — likely a
  timezone-dependent implicit `DateTime → DateTimeOffset` conversion picking up a non-UTC local
  offset on the test runner. Fixed by constructing the `DateTimeOffset` explicitly with
  `TimeSpan.Zero` instead of relying on `.Date`. Worth remembering: prefer explicit
  `new DateTimeOffset(date, TimeSpan.Zero)` over `.Date` when a test needs a UTC-midnight
  timestamp — `.Date` is not timezone-safe.
- Found in passing (not fixed): a proposal created with only a `freeTextModel` vehicle (no
  catalog model) returns `vehicleName: " "` (single space) from `GET /api/proposals` instead of
  the free-text name or `null`. Logged in [[KNOWN-ISSUES]], low priority.

**Didn't work / dead ends:** none this session.

**Links:** [[STATUS]] · [[KNOWN-ISSUES]] · [[GOALS-PERMISSIONS]] · [[CONVENTIONS]] (mobile header
pattern) · [[SECURITY]]
