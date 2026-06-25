# OnTimeCRM — Claude Code Context

Car-dealership CRM SaaS (Portugal). Monorepo:
- `OnTimeCRM_Backend/` — ASP.NET Core 8 + EF Core + PostgreSQL
- `OnTimeCRM_Frontend/` — React 18 + TypeScript + Vite + Ant Design 5
- `ONTIMECRM/` — Obsidian vault: **the source of truth** (read it)

## The vault is the source of truth — read it, keep it updated

Facts (enums, schema, endpoints, business rules) live in `ONTIMECRM/`, NOT in this file.
Do not duplicate them here — they drift.

**At the start of a task:** read the relevant notes.
- Entry point / index: `ONTIMECRM/OnTimeCRM.md`
- Plan & priorities: `ONTIMECRM/01-FEATURES/ROADMAP.md`
- What's done/broken: `ONTIMECRM/00-PROJECT/STATUS.md`
- Entities + canonical enums: `ONTIMECRM/00-PROJECT/DOMAIN.md`
- Patterns + file-size limits: `ONTIMECRM/00-PROJECT/CONVENTIONS.md`
- Layers/auth/middleware: `ONTIMECRM/00-PROJECT/ARCHITECTURE.md`
- Endpoints: `ONTIMECRM/02-DATABASE/API-REFERENCE.md`
- Schema: `ONTIMECRM/02-DATABASE/SCHEMA-*.md`
- Known frontend bugs: `ONTIMECRM/02-DATABASE/FRONTEND-ISSUES.md`

**When anything changes — before ending the session — update the matching note** (table in `ONTIMECRM/00-PROJECT/HOW-TO-USE.md`). If code diverges from a doc, fix the doc. Verify facts against source before trusting a note.

## Non-negotiable rules
1. **Language: English** for all code AND docs. UI text is pt-PT and en-US, served from `/api/i18n` — never hardcoded in components. **Whenever you add or change any UI-facing string** (table header, button, label, placeholder, modal title, validation message, etc.), add the key to **both** `PtPT()` and `EnUS()` in `I18nController.cs`, mirror it in the frontend's `i18nFallback.ts` (pt-PT only), and use `t['YOUR.KEY']` in the component — never a literal string. Before ending a session that touched UI text, grep the changed files for quoted Portuguese literals not wrapped in `t[...]` to catch anything missed, and verify `GET /api/i18n?locale=en-US` returns no fewer keys than `?locale=pt-PT` (a key present in one locale but missing in the other silently falls back to showing the raw key name in the UI for whichever locale is missing it).
2. **Data layer split by complexity:** complex/multi-table/aggregate logic → PostgreSQL stored functions (`fn_*`); simple single-entity CRUD → EF Core in services. Reality today: only 7 read `fn_*` are used, complex writes live in C# services, and ~43 `fn_*` are dead/drifted. See `04-DECISIONS/2026-05-30-data-layer.md` + `00-PROJECT/BEFORE-DEPLOY.md`.
3. **No EF migrations** — `EnsureCreated` + auto drop+recreate on drift. Never run `dotnet ef migrations add`.
4. **`SoldAt` is never auto-set to UtcNow** — always from the request (tests enforce this).
5. **`commission` is private** — never expose in friend-facing endpoints.
6. **Admin role (UserRole=2) bypasses subscription + permission checks.**
7. File-size limits (local model): Controller ~80, Service ~150, Repository ~100, React page ~150, component ~100, doc ~150 lines. Split before exceeding.

## Build / test
```bash
docker-compose up -d                       # API + PostgreSQL
cd OnTimeCRM_Frontend && npm run dev        # localhost:5173
cd OnTimeCRM_Backend && dotnet test         # integration tests (Docker required)
```
Deploy targets: Render.com (API) · Vercel (Frontend) · Supabase (DB).
