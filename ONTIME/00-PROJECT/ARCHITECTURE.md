# OnTime — Architecture

Hub: [[OnTime]] · Code rules: [[CONVENTIONS]] · Domain: [[DOMAIN]].

## What is this?
CRM SaaS for car-dealership salespeople (Portugal). Core value: automatic follow-up notifications triggered by pipeline stage changes. See [[NOTIFICATIONS]].

## Monorepo
```
OnTime_Backend/   → ASP.NET Core 8 API
OnTime_Frontend/  → React 18 + TypeScript + Vite SPA
ONTIME/           → Obsidian vault (docs)
docker-compose.yml   → Local dev (API + PostgreSQL)
```

---

## Backend — layer rules

```
Domain ← Application ← Infrastructure ← API
```
| Layer | Contains | Must NOT contain |
|-------|----------|------------------|
| Domain | Entities, Enums | Business logic, DB |
| Application | Services, Interfaces, DTOs | EF Core, HTTP |
| Infrastructure | Repositories, DbContext, SQL functions | Business rules |
| API | Controllers, Middleware, DI | Business logic |

**File-size targets:** Controller ~80 · Service ~150 · Repository ~100 lines. Split before exceeding — the local model loses context on large files. See [[CONVENTIONS]].

---

## Data layer
Rule + rationale live in [[2026-06-29-data-layer-migrated-to-csharp]] (supersedes [[2026-05-30-data-layer]]). TL;DR: ALL query/aggregate/write logic lives in C#/EF Core now — no PostgreSQL `fn_*` stored functions remain, `DatabaseFunctions.cs` is deleted, nothing is applied at startup beyond `EnsureCreated` + drift-detect.

---

## Pagination — audited 2026-06-29, confirmed already correct
Every `*FilterParams`/`VehicleSearchParams` record defaults to `Page=1, PageSize=20`, and every
repository clamps the incoming value server-side with `Math.Clamp(p.PageSize, 1, 50)` before
querying — a client cannot request `pageSize=999999` and pull a whole table. Confirmed across
`ClientRepository`, `ProposalRepository`, `SaleRepository`, `NotificationRepository`,
`VehicleRepository`. All read-list queries use `.AsNoTracking()` and the standard
count-then-skip/take pattern in C#/EF Core LINQ (see [[2026-06-29-data-layer-migrated-to-csharp]] —
no SQL stored functions anymore). Frontend's `usePagination()` hook defaults to the same 20, matching the
backend. The handful of genuinely unpaginated reads (vehicle brands, a company's own Filiais,
pipeline stages, memberships) are all naturally small/scoped lists, not a risk.
Non-critical follow-up if it ever matters: no composite index on `user_vehicle_models
(user_id, vehicle_brand_id)` — fine while that table stays small per user.

---

## Tenant scoping — audited 2026-06-29, confirmed already correct
Every tenant-scoped entity carries only a Guid FK back to `CompanyId`/`BrandId` — directly
(`LeadSourceOption.CompanyId`, `BrandVehicleBrand.BrandId`) or transitively via `User.CompanyId`/
`BrandId` (`Client`, `Proposal`, `Sale`, `ClientStage`, ...). **No entity stores a denormalized
`CompanyName`/`BrandName` string** — renaming a Company (admin) or Brand (`BrandService.UpdateAsync`)
takes effect everywhere immediately, since every read (all in C#/EF Core LINQ now — see
[[2026-06-29-data-layer-migrated-to-csharp]]) JOINs to the canonical `companies`/`brands` tables at
query time. The JWT itself only ever carries `cid`/`bid` (IDs), never names.

The one place a name is held client-side as a snapshot: `LoginResponseDto.CompanyName`/`BrandName`,
captured at login/switch-brand time into `authStore` — purely cosmetic (sidebar header), never used
for scoping or authorization, and refreshes on next login/switch. Intentionally-global,
not-tenant-scoped tables: `VehicleBrand`/`VehicleModel` (shared catalog "shelf" — per-Filial/per-user
cloning happens downstream, see [[USER-BRANDS]]), `TranslationEntry` (i18n catalog).

## Schema management
**No migrations.** `EnsureCreated` + auto drop+recreate on drift.
- Never run `dotnet ef migrations add`.
- Add entity/column → update entity + EF config + sentinel check in `DatabaseInitializer.IsSchemaCurrentAsync`.
- Restart API → DB auto-recreates.

## Auth
JWT Bearer, `ManagerOnly` policy (role 1 or 2). Full details + hardening backlog in [[SECURITY]]. Roles in [[DOMAIN]].

**Multi-Filial membership (2026-06-27).** A user can belong to several companies/filiais
(`UserBrandMembership`), but `User.CompanyId`/`BrandId` and the JWT's `cid`/`bid` claims stay
**singular** — they're the "currently active" Filial. `AccessScope`/`ManagerBrandScope`/
`RequireBrandId()` are completely unaffected by this — they only ever read the active claim.
Switching (`POST /api/users/me/switch-brand`) re-mints a fresh JWT scoped to the new Filial, same
shape as login. See [[USER-BRANDS]] and `04-DECISIONS/2026-06-27-per-user-vehicle-catalog.md`.

## Error handling
- Business errors → `throw new ApiException(ApiErrorCatalog.CODE)`.
- `ErrorHandlingMiddleware` serializes `{ code, message, class, traceId }`. Postgres `RAISE EXCEPTION 'CODE'` is caught and mapped.

## Subscription middleware
| AccountStatus | Condition | GET | Writes |
|---------------|-----------|-----|--------|
| Active | — | ✅ | ✅ |
| PendingActivation | `TrialEndsAt > now` | ✅ | ✅ |
| PendingActivation | `TrialEndsAt ≤ now` | ❌ 402 | ❌ 402 |
| Expired | — | ✅ | ❌ 402 |
| Suspended | — | ❌ 402 | ❌ 402 |
| Cancelled / Inactive | — | ❌ 403 | ❌ 403 |

Bypass: `/api/auth/*`, `/api/i18n`, `/health`, `/api/subscription/*`, `/api/webhooks/*`. Admins bypass entirely.

**Frontend lockdown for `Expired` (2026-06-29).** The table above already makes `Expired` read-only
on the backend (GET ok, writes 402) — `ProtectedRoute.tsx` now additionally redirects every route
*except* `/profile` to `/profile` when `user.accountStatus === 2` and the user isn't Admin, and
`Sidebar.tsx`'s main nav collapses to empty in that state (Profile stays reachable via the
user-menu dropdown, which is separate from the main `Menu`). Both checks read `accountStatus`
straight off the cached `LoginResponseDto` — no extra API call.

---

## Frontend
- React 18 + TypeScript (strict) + Vite · Ant Design 5 + TailwindCSS
- Zustand: auth/i18n/theme · TanStack Query v5: server state
- react-hook-form + zod · Axios (401→login, 402→subscription, 403→toast)
- Brand color from JWT → `ConfigProvider.token.colorPrimary`
- Dark/light: `themeStore.isDark` → Ant `darkAlgorithm`/`defaultAlgorithm`

Frontend file-size targets are in [[CONVENTIONS]]; context recipes for the local model in [[LOCAL-AI-SETUP]].

---

## Deploy
API → Render.com · Frontend → Vercel · DB → Supabase PostgreSQL

## Local dev
```bash
docker-compose up -d
cd OnTime_Frontend && npm run dev   # localhost:5173
```

---

## Related
[[CONVENTIONS]] · [[DOMAIN]] · [[API-REFERENCE]] · [[SCHEMA-PIPELINE]] · [[OnTime]]
