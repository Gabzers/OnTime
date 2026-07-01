# Manual QA Regression Checklist

Hub: [[OnTime]] · Automated suite: [[TESTING]] · Style/UI rules: [[CONVENTIONS]].

**Purpose:** a running list of flows that have broken before (or are structurally likely to break)
when *adjacent* code changes — not a full test plan, and not a replacement for `OnTime.Tests`. Use
this to scope a fast, targeted manual pass instead of re-testing the whole app from scratch every
time. Add a new entry whenever a bug is found manually that the automated suite didn't catch —
that's a signal the flow deserves a permanent line here (and ideally a new automated test too).

## How to run a pass

1. `docker compose build api frontend && docker compose up -d --force-recreate api frontend` —
   **never test against production** (`getontime.vercel.app`) for anything destructive; register a
   throwaway test account against local docker instead.
2. Register a brand-new Manager (fresh Company + Stand) — don't reuse an old test account, some
   bugs only show up on first-run state (empty catalog, zero `BrandVehicleBrand` rows, etc.).
3. Walk the checklist below top to bottom. Check off what's covered by your change, skim the rest.
4. `dotnet test` (backend) + `npx tsc -p tsconfig.app.json --noEmit` + `npm run build` (frontend)
   — always, regardless of what you touched. Catches regressions this checklist won't.

## Known tooling limitation

Browser automation (Claude in Chrome MCP) has proven unreliable in this environment for:
antd `Select`/`AutoComplete` dropdowns opening/selecting via simulated clicks (works inconsistently
— keyboard search + Enter is more reliable than clicking an option ref), and true mobile-viewport
emulation (`resize_window` doesn't reliably change the actual rendered viewport; forcing
`window.innerWidth` via JS + dispatching a `resize` event is the working fallback). See also the
"Open idea" note in [[BEFORE-DEPLOY]] about giving Claude read-only DB access to make QA validation
less dependent on fragile UI automation — revisit that idea before assuming future QA passes will
be smoother.

---

## Registration & first-run state

- [ ] Register a Manager → Company + Stand + Admin user created atomically, default `IsAutomotive: true`.
- [ ] **Fresh Stand has zero `BrandVehicleBrand` rows on purpose** — the vehicle-brand picker in
      "Nova Proposta" and the Veículos page correctly show **nothing** sellable until the Manager
      explicitly configures brands in Stands → "Marcas que Vendo". This is deliberately restrictive
      (`BrandVehicleBrandsFlowTests.CreateModel_ForBrandNotAllowedByStand_IsRejected`) — don't
      "fix" it back to permissive without checking that test first.
- [ ] Default Lead Sources (Stand/Telefone/Instagram/Facebook/Recomendação/Outro) seeded for the new user.
- [ ] Default Pipeline stages seeded (7 stages, first one active, Won/Lost at the end).

## Client → Proposal → Sale (the core CRM loop)

- [ ] Create a client (no proposal) — lands in the first Pipeline stage, temperature Quente.
- [ ] Configure at least one vehicle brand on the Stand before creating a proposal, or "Modelo"
      stays empty forever (see "first-run state" above — not a bug, just a prerequisite).
- [ ] Create a proposal for that client, with a vehicle from the (now-configured) catalog.
- [ ] Convert the proposal to a Sale — `SoldAt` **must** be a field you fill in, never silently
      today's date (`CLAUDE.md` rule, has its own regression test — `SoldAt` auto-set is the one
      thing that must never regress silently).
- [ ] Client moves to the Won stage automatically; a post-sale follow-up notification is scheduled.
- [ ] Mark a *different* proposal as Lost — client moves to Lost stage, reason/notes saved.
- [ ] Delete a Lost/Active proposal → soft-deletes, disappears from every list. Delete a **Won**
      proposal → blocked (`PROPOSAL_ALREADY_CLOSED`), UI shows the disabled delete button with a
      tooltip explaining why.
- [ ] Delete a client → soft-deletes, disappears from the Clients list, but their sales/proposals
      stay in the DB untouched (check via `/api/admin` or DB directly — not visible in the UI once
      the client itself is gone, that's the whole point of soft delete here).

## Mobile vs desktop parity

- [ ] Every `ResourcePage`-based list (Clients, Proposals, Sales, Vehicles, Brands, Lead Sources)
      — check the **mobile card view** has the same action buttons as the desktop table. This has
      broken twice already (Brands/Lead Sources missing edit; Clients missing delete) from a
      copy-paste gap between `columns` and `renderMobileCard` — it's an easy thing to forget when
      adding a new action to a table.
- [ ] No bottom tab bar on mobile (removed 2026-07-01) — only the hamburger drawer for navigation.
- [ ] Sidebar drawer (mobile) lists every nav item the desktop sidebar does.

## Delete confirmations

- [ ] Every delete/remove button anywhere in the app shows a `Popconfirm` before firing — none
      should delete on a single click. If you add a new delete action, use `DeleteWithConfirm`
      (`components/generic/DeleteWithConfirm.tsx`), never wire the mutation directly to `onClick`.
      See [[CONVENTIONS]] "Every destructive action must confirm first".

## Goals

- [ ] Create a goal (any metric/period/target).
- [ ] **Edit an existing goal and change the metric type and period, not just the target value** —
      all 4 fields (`MetricType`, `Period`, `TargetValue`, `ShowOnDashboard`) must be editable and
      persist. This was broken until 2026-07-02 (edit form silently hid metric/period).
  - [ ] Switching a goal *to* ConversionRate re-applies the ≤100 cap even if the target was already
        set higher under a different metric.
- [ ] Reorder goals via the arrow buttons — order persists and reflects on the Dashboard's pinned
      goals widget too.
- [ ] Delete a goal — confirmation required, soft-deletes.

## Notifications / stage automation

- [ ] Configure a stage's temperature rules (StageConfigDrawer) — a client sitting in that stage
      past the configured day threshold changes temperature automatically. This only actually
      fires via the `pg_cron`-triggered `/api/internal/run-scheduled-jobs` job (every 15 min in
      prod) — don't expect it to happen instantly, and don't test it by waiting on production;
      call the internal endpoint directly (with the secret, never pasted into a browser script —
      see [[SECURITY]]) or trigger `IScheduledJobsService.RunAsync` directly in a test.
- [ ] Configure a recurring notification template on a stage — fires on schedule, stops when the
      client changes stage (regardless of `MaxOccurrences`).
- [ ] Digest email and Business Summary email toggles in Profile — each independently
      enable/disable-able, respect the recipient's `Locale`, not the sender's.

## i18n / theming

- [ ] Toggle dark/light mode — check a page you didn't touch too, not just the one you changed
      (theme tokens sometimes leak a hardcoded color that only shows in one mode).
- [ ] Switch language pt-PT ↔ en-US — check `GET /api/i18n?locale=X` returns the same key **count**
      for both locales before trusting the UI (a key present in one locale and missing in the
      other silently falls back to showing the raw key name for whichever locale is missing it).
- [ ] If you added/changed any UI-facing string this session, grep the changed files for quoted
      Portuguese literals not wrapped in `t[...]` — this is a CLAUDE.md non-negotiable, not optional.

## Related
[[TESTING]] · [[CONVENTIONS]] · [[KNOWN-ISSUES]] · [[SECURITY]] · [[BEFORE-DEPLOY]] · [[OnTime]]
