# Features: Goals & Access Control

Hub: [[OnTimeCRM]] · Schema: [[SCHEMA-CONFIG]] · API: [[API-REFERENCE]].

Both are **implemented** (controller + service + DB). Status: [[STATUS]].

---

## Goals (UserGoal) — implemented

Personal performance targets, not shared. Progress computed live from dashboard KPIs (`UserGoalService` calls `ISaleService.GetDashboardAsync`).

- `MetricType`: NewClients=0, Sales=1, Proposals=2, ConversionRate=3
- `Period`: Daily=0, Weekly=1, Monthly=2
- `TargetValue`, `StartDate`, `EndDate?`
- Delete = soft delete (`IsActive=false`). Ownership enforced (`AUTH_FORBIDDEN` if not owner).

Progress mapping (current value vs target):
- Sales → `dashboard.SalesThisMonth`
- Proposals → `dashboard.ProposalsThisMonth`
- NewClients → `dashboard.ActiveClients`
- ConversionRate → `dashboard.ConversionRate`

Endpoints (scoped to JWT userId):
```
GET    /api/goals          list with live progress
POST   /api/goals          create
PUT    /api/goals/{id}     update target/dates
DELETE /api/goals/{id}     soft delete
```

---

## Access Control (MenuItemPermission) — implemented

UI-level permissions per role per route. **API auth is still enforced server-side** by `[ManagerOnly]`/`[Authorize]` regardless of these flags.

- Seeded on first call (`EnsureSeedAsync`):
  - Manager (1): all flags true on all routes
  - Salesperson (0): `/brands`, `/admin`, `/access-control` → all false; others true
  - **Admin (2): full access computed in-memory, never stored**
- Routes seeded: `/dashboard /clients /proposals /sales /notifications /stages /vehicles /goals /friends /brands /admin /access-control`

> ⚠️ Reality vs intent: the entity has 4 flags (`CanView/CanCreate/CanEdit/CanDelete`), but the **API exposes only 2** (`CanRead`, `CanEdit`). On update, `CanEdit` is copied into Create+Edit+Delete. The DTO is `MenuPermissionDto(Id, Role, RouteKey, CanView, CanEdit)`.

Endpoints (manager-only):
```
GET /api/permissions?role={int}     get permissions for a role
PUT /api/permissions/{role}         body: UpdateMenuPermissionRequest[] (RouteKey, CanRead, CanEdit)
```

Frontend `usePermissions` hook drives nav visibility.

## Related
[[SCHEMA-CONFIG]] · [[DOMAIN]] · [[FRIENDS]] · [[DASHBOARD]] · [[OnTimeCRM]]
