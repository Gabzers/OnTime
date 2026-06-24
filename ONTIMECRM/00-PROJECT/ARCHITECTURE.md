# OnTimeCRM — Architecture

Hub: [[OnTimeCRM]] · Code rules: [[CONVENTIONS]] · Domain: [[DOMAIN]].

## What is this?
CRM SaaS for car-dealership salespeople (Portugal). Core value: automatic follow-up notifications triggered by pipeline stage changes. See [[NOTIFICATIONS]].

## Monorepo
```
OnTimeCRM_Backend/   → ASP.NET Core 8 API
OnTimeCRM_Frontend/  → React 18 + TypeScript + Vite SPA
ONTIMECRM/           → Obsidian vault (docs)
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
Rule + current reality + rationale all live in [[2026-05-30-data-layer]]. TL;DR: complex/aggregate → `fn_*`; simple CRUD → EF. Today only 7 read `fn_*` are used; ~43 are dead.

---

## Schema management
**No migrations.** `EnsureCreated` + auto drop+recreate on drift.
- Never run `dotnet ef migrations add`.
- Add entity/column → update entity + EF config + sentinel check in `DatabaseInitializer.IsSchemaCurrentAsync`.
- Restart API → DB auto-recreates.

## Auth
JWT Bearer, `ManagerOnly` policy (role 1 or 2). Full details + hardening backlog in [[SECURITY]]. Roles in [[DOMAIN]].

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
cd OnTimeCRM_Frontend && npm run dev   # localhost:5173
```

---

## Related
[[CONVENTIONS]] · [[DOMAIN]] · [[API-REFERENCE]] · [[SCHEMA-PIPELINE]] · [[OnTimeCRM]]
