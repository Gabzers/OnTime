# Data layer: all logic now in C#/EF Core — SQL stored functions retired

Date: 2026-06-29 · [[OnTime]] · supersedes [[2026-05-30-data-layer]]

**Rule (new):** ALL query/aggregate/write logic lives in C#/EF Core (LINQ in repositories or services). No PostgreSQL stored functions (`fn_*`) remain in the codebase or are applied at startup.

**Why:** the 2026-05-30 split (complex → SQL functions, simple → C#) never held in practice — only 7 of 56 `fn_*` functions were ever called, the other 49 were dead code re-applied every startup, and several had drifted from the C# truth (e.g. `fn_register_manager` set `account_status=Active` while `AuthService` set `PendingActivation`). Discussed with the user: a well-written EF Core LINQ query compiles to one SQL round-trip, same as a stored-function call, so there's no real performance cost to dropping SQL functions on Supabase/Render free tiers — the only cost driver would be badly-written C# doing N+1 queries, not the language choice. Splitting logic across two languages added maintenance and drift risk for no measurable benefit, so the user opted for full consistency: everything in C#.

**What changed:**
- `ClientRepository.GetPagedAsync` / `GetHotDealsAsync` — `fn_get_clients_paged` / `fn_get_hot_deals` → LINQ.
- `ProposalRepository.GetPagedAsync` — `fn_get_proposals_paged` → LINQ (preserves the two-step COALESCE vehicle-name logic: preferred vehicle's catalog "Brand Model" if set, else its free-text model).
- `NotificationRepository.GetTodayAsync` — `fn_get_today_notifications` → LINQ.
- `SaleRepository.GetDashboardAsync` — `fn_get_dashboard_kpis` + `fn_get_monthly_stats` + `fn_get_loss_reasons` → LINQ (KPIs, 6-month zero-filled series, loss-reason counts, top-5 hot deals all computed in one repository method).
- `DatabaseFunctions.cs` (1958 lines, 56 `CREATE OR REPLACE FUNCTION` definitions) deleted entirely.
- `DatabaseInitializer.InitializeAsync` no longer drops/applies any SQL functions at startup — just `EnsureCreated` + drift-detect + the existing `/admin` menu-permission cleanup.
- `OnTime_Backend/sql/functions/*.sql` (10 files) were already a dead, unloaded mirror before this change (confirmed via exhaustive grep — nothing in C# ever read them) — left in place as-is, flagged as stale and safe to delete in a future cleanup pass.

**Gotcha hit during migration:** `GroupBy(...).Select(g => new Dto(...)).OrderByDescending(d => d.SomeProjectedProperty)` does not translate to SQL in EF Core 8/Npgsql — ordering by a property of an already-projected DTO inside the same query causes a runtime `InvalidOperationException` at query-compile time, not a build error. Fix: materialize the grouped result with `.ToListAsync()` into an anonymous type first, then `.OrderByDescending(...).Select(...)` in-memory (LINQ-to-Objects) for the final DTO shape. See `SaleRepository.GetDashboardAsync`'s loss-reasons block.

**Verification:** full `dotnet test` suite green (192/192) after migration; `docker compose build api && docker compose up -d --force-recreate api` confirmed the container starts cleanly with no SQL-function startup step and serves `/api/i18n` correctly.

Affects: [[ARCHITECTURE]] · [[CONVENTIONS]] · [[STATUS]] · [[ROADMAP]]
