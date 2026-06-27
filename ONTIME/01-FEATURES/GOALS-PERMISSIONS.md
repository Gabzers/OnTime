# Features: Goals & Access Control

Hub: [[OnTime]] · Schema: [[SCHEMA-CONFIG]] · API: [[API-REFERENCE]].

Both are **implemented** (controller + service + DB). Status: [[STATUS]].

---

## Goals (UserGoal) — implemented

Personal performance targets, not shared. **Progress is computed from the goal's own date
range** (`UserGoalService.ComputeCurrentValueAsync`, rewritten 2026-06-25) — counts
sales/proposals/new-clients directly from the DB for `[StartDate, EndDate ?? PeriodEnd)`, never
a shared "this calendar month" snapshot. This matters: a Daily/Weekly/Annual goal with the old
implementation always showed the same number as a Monthly one, because every period reused
`dashboard.SalesThisMonth` etc.

- `MetricType`: NewClients=0, Sales=1, Proposals=2, ConversionRate=3
- `Period`: Daily=0, Weekly=1, Monthly=2, **Annual=3** (added 2026-06-25)
- `TargetValue`, `StartDate`, `EndDate?` (backend-only now — see below), `ShowOnDashboard`
  (pins the goal's progress bar to the Dashboard home screen; see `DashboardPage.tsx`)
- **Frontend no longer collects `EndDate` at all** (2026-06-25) — the create/edit form only asks
  for `StartDate`, defaulted to the start of the chosen `Period` (`startOfPeriod()` in
  `GoalsPage.tsx`). Previously defaulted to "today", so a Monthly goal created mid-month
  silently excluded every sale from earlier that month. `EndDate` stays in the API/DB purely as
  an internal override hook (always null from the UI); `ComputeCurrentValueAsync` falls back to
  `PeriodEnd(period, start)` whenever it's null, which is now always.
- Delete = soft delete (`IsActive=false`). Ownership enforced (`AUTH_FORBIDDEN` if not owner).

Progress per metric (all scoped to the goal's own window, see above):
- Sales → count of `Sale` rows with `SoldAt` in range
- Proposals → count of `Proposal` rows with `ProposalDate` in range
- NewClients → count of `Client` rows with `CreatedAt` in range (was, incorrectly, lifetime
  `dashboard.ActiveClients` before the rewrite)
- ConversionRate → sales-in-range / proposals-in-range × 100, 0 if no proposals in range

Endpoints (scoped to JWT userId):
```
GET    /api/goals          list with live, period-scoped progress
POST   /api/goals          create
PUT    /api/goals/{id}     update target/dates/ShowOnDashboard
DELETE /api/goals/{id}     soft delete
```

---

## Access Control (MenuItemPermission) — implemented

UI-level permissions per role per route. **API auth is still enforced server-side** by `[ManagerOnly]`/`[Authorize]` regardless of these flags.

- Seeded on first call (`EnsureSeedAsync`):
  - Manager (1): all flags true on all routes
  - Salesperson (0): `/brands`, `/access-control` → all false; others true
  - **Admin (2): full access computed in-memory, never stored**
- Routes seeded: `/dashboard /clients /proposals /sales /notifications /stages /vehicles /goals /friends /brands /access-control`
- **`/admin` is deliberately NOT in this list** (2026-06-25) — it used to be, which meant a
  Manager's own per-company permission screen had a toggle implying cross-tenant platform-admin
  access was something a company could grant itself. Fixed alongside the `AdminOnly` policy fix
  — see [[SECURITY]]. A startup cleanup (`DatabaseInitializer`) deletes any pre-existing
  `/admin` row left over from before this change.

All 4 flags (`CanView/CanCreate/CanEdit/CanDelete`) are independent end-to-end — fixed T5
(2026-06-06), `MenuPermissionDto`/`UpdateMenuPermissionRequest` both expose all 4.

Endpoints (manager-only):
```
GET /api/permissions?role={int}     get permissions for a role
PUT /api/permissions/{role}         body: UpdateMenuPermissionRequest[] (RouteKey, CanView, CanCreate, CanEdit, CanDelete)
```

Frontend `usePermissions` hook drives nav visibility.

## Related
[[SCHEMA-CONFIG]] · [[DOMAIN]] · [[FRIENDS]] · [[DASHBOARD]] · [[OnTime]]
