# Data layer: complex logic in stored functions, simple CRUD in services

Date: 2026-05-30 · [[OnTime]]

**Rule:** complex / multi-table / set-based logic lives in PostgreSQL stored functions (`fn_*`); simple single-entity CRUD stays in C# services using EF Core.

**Why:** push heavy queries and aggregation to the DB (one round-trip, Render/Supabase latency), keep trivial reads/writes simple and testable in C#.

**Implies (do / don't):**
- ✅ paginated lists, dashboard aggregates, search → `fn_*` (called via `db.Database.SqlQuery<T>()`)
- ✅ simple find / create / update of one entity → EF + `SaveChangesAsync` in the service
- ✅ when adding complex logic, prefer a new `fn_*` over fat C# loops
- ❌ don't reimplement the same complex flow in both places

**Current reality (verified 2026-05-30) — diverges from the rule:**
- **Used `fn_*` (7, all reads):** `fn_get_clients_paged`, `fn_get_hot_deals`, `fn_get_proposals_paged`, `fn_get_today_notifications`, `fn_get_dashboard_kpis`, `fn_get_monthly_stats`, `fn_get_loss_reasons`.
- **Complex write flows currently live in C# services**, not in SPs: registration+seeding (`AuthService`), client create + first proposal (`ClientService.CreateAsync`), stage change with snapshot + notification generation (`ClientService.UpdateStageAsync`), convert-to-sale (`ProposalService.ConvertToSaleAsync`).
- **~43 `fn_*` are dead code** — defined in `DatabaseFunctions.cs` and re-applied every startup, never called. Several have **drifted** from the C# truth (e.g. `fn_register_manager` sets `account_status=1 Active`; `AuthService` sets `PendingActivation`).

**Action (pre-deploy — see [[BEFORE-DEPLOY]]):** for each complex write flow decide its home per the rule (SP vs service), then delete the unused/drifted duplicates so there is exactly one implementation.

Affects: [[ARCHITECTURE]] · [[CONVENTIONS]] · [[NOTIFICATIONS]] · [[STATUS]]
