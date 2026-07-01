# Features: Goals & Access Control

Hub: [[OnTime]] · Schema: [[SCHEMA-CONFIG]] · API: [[API-REFERENCE]].

Both are **implemented** (controller + service + DB). Status: [[STATUS]].

---

## Goals (UserGoal) — implemented

Personal performance targets, not shared. **`UserGoal` has no `StartDate`/`EndDate` fields at
all** (removed since this doc was last accurate) — progress is always computed for the
**current** occurrence of the goal's `Period` (`GoalPeriodCalculator.Window(period, now)`): this
week/month/year, no fixed dates to pick or drift out of sync with. A Monthly goal always reports
this calendar month's numbers; a Weekly goal always reports Mon-Sun of the current week; etc.

- `MetricType`: NewClients=0, Sales=1, Proposals=2, ConversionRate=3
- `Period`: Daily=0, Weekly=1, Monthly=2, Annual=3
- `TargetValue`, `ShowOnDashboard` (pins the goal's progress card to the Dashboard home screen),
  `SortOrder` (manual drag/arrow reorder, also drives Dashboard pinned-goal order)
- **Every field is editable after creation** (fixed 2026-07-02 — previously `PUT /api/goals/{id}`
  only accepted `TargetValue`/`ShowOnDashboard`; `MetricType`/`Period` were create-only and the
  edit form hid those fields entirely, so fixing a typo'd metric meant delete-and-recreate,
  losing the goal's `SortOrder`/history). `ConversionRate`'s ≤100 cap is now validated against
  the *new* `MetricType` in the request, not the goal's old one — switching a goal to/from
  ConversionRate re-evaluates the cap correctly either direction.
- Delete = soft delete (`IsActive=false`). Ownership enforced (`AUTH_FORBIDDEN` if not owner).

Progress per metric (all scoped to the current period window, see above):
- Sales → count of `Sale` rows with `SoldAt` in range
- Proposals → count of `Proposal` rows with `ProposalDate` in range
- NewClients → count of `Client` rows with `CreatedAt` in range
- ConversionRate → sales-in-range / proposals-in-range × 100, 0 if no proposals in range

Endpoints (scoped to JWT userId):
```
GET    /api/goals          list with live, period-scoped progress
POST   /api/goals          create
PUT    /api/goals/{id}     update — metric, period, target, and pin flag, all of it
DELETE /api/goals/{id}     soft delete
PUT    /api/goals/reorder  bulk SortOrder update from an ordered id list
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
