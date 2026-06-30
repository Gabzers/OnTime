# 🎯 ROADMAP

Ordered plan of attack. Hub: [[OnTime]] · Detailed state: [[STATUS]].

> **Rule:** when an item is done, tick it `[x]` here and update [[STATUS]].
> Significant decision during a phase → add a note in `04-DECISIONS/` (see [[_TEMPLATE]]).
> **For the local AI:** execute via [[AI-TASK-PLAN]] using the test-first loop in [[AI-WORKFLOW]].

The backend is more complete than it first looked: Goals, Friends, Permissions and
Subscription endpoints all exist. The real gaps are the frontend bugs, payment
integration, and the i18n system.

---

## Phase 0 — Stabilize the frontend ✅ done (2026-06-06)

All T1–T15 tasks complete (see [[AI-TASK-PLAN]] for the full TDD log). 128 backend + 15 frontend
tests green. Highlights: snooze field fix, i18n label swap fix, trial lockout fix, hardcoded
strings removed, FOUC fix, missing i18n keys, proposal-detail back-nav, proposal vehicle
requirement, vehicle status dot/toggle. Full bug/debt catalog in [[KNOWN-ISSUES]]; deploy-gating
infra in [[BEFORE-DEPLOY]].

---

## Phase 1 — Close MVP gaps (NOW) — **Sprint 1**

- [x] **F1 — User Brands filter** ✅ 2026-06-06. See [[USER-BRANDS]].
- [ ] **Email verification** flow (post-registration). See [[SCHEMA-AUTH]]. **Deliberately last in
      Sprint 1** (2026-06-27, user's call) — do every other Sprint 1 item before this one.
- [x] Audit the implemented Goals/Friends/Permissions pages end-to-end ✅ 2026-06-25 (full Chrome sweep + test-quality audit — see [[STATUS]]). Found and fixed: Goals date-window bug, Friends email/name-swap bug, the cross-tenant Admin-panel authz hole. See [[GOALS-PERMISSIONS]], [[FRIENDS]], [[SECURITY]].
- [ ] Confirm dashboard performance target (<500ms). See [[DASHBOARD]].
- [x] Data-layer reconciliation — all logic migrated to C#/EF Core, `DatabaseFunctions.cs` retired ✅ 2026-06-29. See [[2026-06-29-data-layer-migrated-to-csharp]].

---

## Phase 1.5 — UX & admin polish (2026-06-26 backlog) — **Sprint 1**

User-requested batch, clarified via `/grill-me` before being recorded here. **Email verification
moved to the end of the sprint** at the user's request (2026-06-27) — do it last, not first.

- [x] **Goals page + home screen visual redesign.** ✅ 2026-06-29 — new shared `GoalCard.tsx`
      component (used by both `/goals` and the Dashboard's pinned-goals widget): gradient backdrop
      tinted by the metric's color, a circular progress ring (`Progress type="circle"`) instead of
      a flat bar, a motivational tag that escalates with progress (`utils/goalHelpers.ts`'s
      `motivationKey` — "Vamos a isso!" → "A ganhar tração!" → "Bom ritmo!" → "Quase lá!" →
      "Objetivo atingido!"), a gold trophy ring + border glow once a goal hits 100%, and a
      "N dias restantes" countdown computed from the goal's period (`daysLeftInPeriod`). `/goals`
      also gained a gradient hero header with a one-line subtitle and an achieved-goals counter
      pill. Fixed a real mistranslation found along the way: `LABEL.GOAL.PINNED` said "★ Início"
      (literally "Start") instead of "Fixado"/"Pinned". No functional/data change — purely visual.
      `tsc --noEmit` clean, i18n key parity confirmed (464/464). Verified in browser (dark mode):
      both the `/goals` grid and the Dashboard's compact widget render the new card correctly.
- [ ] **Daily digest — still to build (flagged 2026-06-29).** `NotificationPreference.DigestEnabled`/
      `DailyDigestTime`/`DigestFrequencyDays` already exist in the data model and are editable from
      `ProfilePage` ("Resumo diário ativo"), but **nothing reads them yet** — there is no background
      job/scheduler anywhere in the backend (confirmed: zero `BackgroundService`/`IHostedService`/cron
      in the codebase) that actually sends a digest. Today the toggle is a no-op. Needs: a spec pass
      (exact content — likely today's pending + overdue notifications, per [[NOTIFICATIONS]] — plus
      delivery channel: email vs in-app) before implementation. See [[NOTIFICATIONS]] for what
      already works (`/api/notifications/today`, `/overdue-count`) vs. what doesn't (the digest
      itself).
- [x] **Optional "already has a proposal" checkbox on client creation.** ✅ 2026-06-27 — `CreateClientRequest.HasProposal` (default `true`); `NewClientPage` has a "Já fez proposta?" switch that conditionally renders the whole Proposta card. See [[STATUS]].
- [x] **Lead Source maintenance screen.** ✅ 2026-06-29 — new `LeadSourceOption` entity
      (`CompanyId`, `Code`, `Name`), company-scoped. `Client.LeadSource` stays a plain `int`
      (referencing `LeadSourceOption.Code`, unique per company) — kept deliberately minimal: no
      change to the SQL paged-list functions (`fn_get_clients_paged` etc.), the filter int param,
      or `ClientDto`/`ClientListDto` shapes. New `/api/lead-sources` (GET open to any authenticated
      user of the company; POST/PUT/PATCH `ManagerOnly`), new `LeadSourcesPage.tsx` (CRUD list,
      mirrors `BrandsPage.tsx`). Every new company is seeded with the original 8 defaults
      (Stand/Telefone/OLX/Standvirtual/Instagram/Facebook/Referência/Outro, codes 0-7) at
      registration; Manager/Admin can rename, add, or deactivate (deactivated ones disappear from
      pickers but historical clients keep their stored code/name). `LeadSourceTag` now renders the
      resolved option `Name` directly instead of a translated `ENUM.LEAD_SOURCE.*` key — lead
      sources are free-text per company now, not a fixed translatable set. Also fixed a real
      pre-existing bug found along the way: `NewClientPage`'s old hardcoded dropdown options array
      was `[0..6]` and silently excluded "Outro" (code 7).
- [x] **"Not an automotive account" toggle.** ✅ 2026-06-29 — landed at the **Filial** level
      (`Brand.IsAutomotive`, default `true`), not a per-user profile setting — consistent with
      today's Filial-centric vehicle-brand config, configured by Manager/Admin on `BrandsPage`'s
      edit form ("Vende veículos?" Switch). `LoginResponseDto`/switch-brand response carry it so
      the frontend has it without an extra round trip. When off: `Sidebar` hides the "Veículos" nav
      item; `NewClientPage`/`ClientDetailPage`/`ProposalDetailPage` hide the vehicle-picker section
      entirely; backend also **drops the "≥1 vehicle required" rule** for that Filial's proposals
      (`ClientService.CreateAsync`/`ProposalService.CreateForClientAsync` check
      `IUserRepository.IsAutomotiveAsync`) — otherwise a non-automotive tenant could never create a
      proposal once the picker is hidden. Takes effect on next login/Filial-switch, not live-pushed
      to an already-open session (same caveat as multi-Filial membership). 5 new tests,
      179/179 backend green. Verified end-to-end in browser.
- [x] **Admin panel: fix broken expanded-row layout.** Already done 2026-06-27 — see [[STATUS]]
      "Admin panel expanded-row layout fixed". This roadmap line was stale; corrected here.
- [x] **Profile: let a user edit their own email + personal data.** ✅ 2026-06-29 — extended
      `UpdateUserRequest`/`PUT /api/users/me` with an optional `Email`, guarded by
      `USER_EMAIL_TAKEN` (409) via `IUserRepository.EmailTakenByAnotherUserAsync`.
      `ProfilePage`'s personal-data form now fetches `usersApi.getMe()` (previously only read the
      cached login-time store, which never had `phone` and left it permanently blank) and gained
      an editable E-mail field. Note: changing email does **not** trigger re-verification — email
      verification itself is still open (deliberately last in Sprint 1, see Phase 1 above).
      3 new tests (`ProfileSelfEditFlowTests.cs`), 182/182 green. Verified in browser: phone
      saved correctly, email round-trips via `/api/users/me`.
- [x] **Part 1/2 done 2026-06-27 — registration role + Profile display.** Verified in source:
      `RegisterManagerAsync` always sets `Role = UserRole.Manager`, `RegisterSalespersonAsync`
      always sets `Role = UserRole.Salesperson` — no code path ever assigns `Admin`, and
      `AuthFlowTests` already asserts the role on both registration paths (regression-safe).
      `ProfilePage` now shows the user's own role as a `Tag` next to the page title
      (`ENUM.USER_ROLE.{role}`).
- [x] **Part 2/2 done 2026-06-27 — Admin panel: platform Admin can change a user's role.**
      `PATCH /api/admin/users/{id}/role` + `GET /api/admin/companies/{companyId}/users`, both
      behind `AdminOnly` (never `ManagerOnly`). Self-protection: `CANNOT_CHANGE_OWN_ROLE` (422) if
      the acting Admin targets their own account — both backend-enforced and the frontend disables
      the Select for that row. UI: `AdminPage`'s expanded company row now shows a "Utilizadores"
      table alongside "Filiais", with an inline role `Select` per user. 3 new tests
      (`AdminFlowTests`: Manager forbidden, Admin can change others, Admin can't change self).
- [x] **Optional license-plate (matrícula) field per vehicle row.** ✅ 2026-06-27 — `ProposalVehicle.Plate`
      end-to-end (entity, DTOs, `VehicleProposalTable` column); convert-to-sale inherits the
      preferred vehicle's plate when `ConvertToSaleRequest.Plate` isn't explicitly given.
- [x] **Vehicle catalog redesigned to be per-user, not global-with-filter.** ✅ 2026-06-27 —
      user-requested mid-session after reviewing how role permissions work. See
      [[2026-06-27-per-user-vehicle-catalog]] and [[USER-BRANDS]] for the full design
      (clone-on-select, hide-on-unselect, delete-blocked-if-in-use). `VehiclesController` model
      CRUD is no longer `ManagerOnly` — every role manages their own catalog now.
- [x] **Vehicle brands moved to per-Filial config + multi-Filial membership.** ✅ 2026-06-27, same
      day — see [[2026-06-27-filial-vehicle-brands-and-membership]] and [[USER-BRANDS]]. Planned
      via `EnterPlanMode` given the size/risk (touches auth claims, tenant isolation). Manager/Admin
      now configure which car brands a Filial sells (`BrandVehicleBrand`); personal catalog
      ownership stays per-user, cloning is now lazy. New `UserBrandMembership` lets a user belong
      to several companies/filiais and switch the active one from the sidebar.
- [x] **Configured-only vehicle selection.** ✅ 2026-06-27 — a model without at least one version
      (not "configured"/green) can no longer be picked in a proposal vehicle picker, regardless of
      its `IsActive` flag. See [[CONVENTIONS]] "Configured-only vehicle selection".
- [x] **Tables must have search + sort, standardized.** ✅ 2026-06-27 — new app-wide rule in
      [[CONVENTIONS]]. Applied to `VehiclesPage` (model sort), `AdminPage` (companies/users/brands
      sub-tables gained search+sort). Removed the now-unneeded client-side search box from the
      Dashboard's Hot Deals widget (small, curated list — search was noise there).
- [x] **Manual end-to-end browser verification of the Filial/membership redesign.** ✅ 2026-06-29 —
      walked every new piece live (brand config save, lazy clone, brand filter, configured-only
      picker, Filial switcher, isolation between two Filiais, Admin "Atribuir Filial"). Found and
      fixed a real bug along the way: `MSG.EMPTY_STATE.VEHICLES` missing from both i18n locales.
- [x] **"Configurado" filter on Veículos, defaults to configured-only.** ✅ 2026-06-29 — see
      [[VEHICLES]]. New `VehicleSearchParams.Configured`; `VehiclesPage` filter bar gained a
      Select defaulting to "Configurados" so the management screen itself opens scoped to
      sellable models, matching the proposal-picker rule above.
- [x] **Pagination/query-efficiency audit.** ✅ 2026-06-29 — confirmed already correct app-wide
      (20/page default, 50 server-side cap, `.AsNoTracking()`, count+skip/take). See
      [[ARCHITECTURE]] "Pagination". No changes needed.
- [ ] **Configurable lead-temperature states with time-based auto-transitions.** Requested
      2026-06-27, to be designed via `/grill-me` before implementation — still Sprint 1 per the
      user's explicit timing, but the interview hadn't been completed yet when this was last
      updated. Replaces the hardcoded Hot/Warm/Cold in `ClientService.RecalcTemperature` (see
      Phase 4's existing "Configurable lead-temperature rules" entry below — this supersedes it
      with an active, in-progress design rather than a future idea).

**Also found while documenting (not requested, flagged proactively):**
- [x] **Fixed immediately (2026-06-26):** the new Stages-page notification Popover (bell icon, see
      [[STATUS]] "single-color auto-fill" entry) used `trigger="hover"` only, which never opens on
      touch devices (no hover state) — changed to `trigger={['hover', 'click']}`.

---

## Phase 2 — Monetization — **Sprint 2, deliberately deferred (2026-06-27)**

Everything else in this roadmap (Phase 1/1.5) is Sprint 1. Payments specifically is NOT part of
Sprint 1 — explicitly held back by the user until the rest of the backlog is done.

- [ ] **Stripe** — checkout + webhook. See [[PAYMENTS]].
- [ ] **Ifthenpay** — MBWay + Multibanco + callback. See [[PAYMENTS]].
- [ ] Subscription page wired to real `InitiateAsync` (currently throws `PAYMENT_PENDING`).
- [ ] **Expired-account flow needs a real path forward, not just a message (flagged 2026-06-29).**
      Right now, an expired user sees "contacte o administrador" on `/profile` and that's the end
      of it — no in-app way to actually reach an admin, no renewal CTA, no notification to the
      Manager/Admin that someone on their team is locked out. Once Stripe/Ifthenpay land, this
      needs: a real "Renew" button on the expired-state message (linking into the same
      `InitiateAsync` flow), and ideally a notification/email to the company's Manager/Admin when
      a teammate's subscription expires, so they don't find out from the teammate complaining.
      Deliberately deferred — noted here so it doesn't get lost.

→ Depends on Phase 0. Unblocks real launch.

---

## Sprint 2/3 — Filial join-request workflow (spec captured 2026-06-29, not started)

**Today:** registering a Manager *always* creates a brand-new Company + Filial — there's no way
to join an existing one at sign-up. A Filial is mandatory at account creation (the app doesn't
function without one). Afterwards, switching Filial only works via the existing
`UserBrandMembership`/`switch-brand` mechanism, and membership itself can only be **granted** by a
Manager/Admin (`POST /api/brands/{id}/members`, `POST /api/admin/users/{id}/memberships`) — there
is no self-service way for a user to request to join a Filial they don't already have a grant for.
Any authenticated/active user can already switch freely between every Filial they have membership
in — there's no extra restriction today beyond "do you have a membership row."

**Wanted (user's words, 2026-06-29):** at registration, the user must either (a) create their own
new Filial (stand), same as today, or (b) pick an existing Filial and **send a join request**. The
account stays restricted — can't use the app — until one of three things happens:
1. The join request is **accepted** (by that Filial's Manager/Admin).
2. The user **withdraws/changes** the pending request (e.g. targets a different Filial instead).
3. The user creates their **own** new Filial instead of waiting.

**Today's partial workaround:** an `Admin` (platform role) can already switch into *any* Filial via
the existing membership/switch-brand mechanism (no request needed — Admin bypasses
permission/subscription checks everywhere, per CLAUDE.md rule #6) — everyone else can only get into
a Filial they were explicitly granted into by that Filial's own Manager, or create a new one.

**Not designed yet** — needs its own spec pass before implementation: a `BrandJoinRequest` entity
(`UserId`, `BrandId`, `Status: Pending/Accepted/Rejected/Withdrawn`), a "blocked" account state distinct
from today's `PendingActivation`/subscription gating, a Manager-side inbox to accept/reject incoming
requests, and a decision on whether a user can have more than one pending request at once (the
"or change the request" wording suggests no — withdrawing the old one is required before sending a
new one). Revisit before starting Phase 2 (Monetization) work, since the "does this account even
have a usable Filial yet" question affects subscription/trial gating too.

---

## Phase 3 — i18n system rebuild

Translations are hardcoded in `I18nController.cs` (pt-PT + en-US, both now genuinely complete —
2026-06-25, see [[I18N]]); `TranslationEntry` is dead. Remaining work here is purely the
architecture decision below, not missing/broken translations.

- [ ] Decide: keep hardcoded dictionaries, or move to DB (`translation_entries` + `translation_locales`)?
- [ ] Missing-key detection
- [ ] Optional auto-translation via external API (DeepL/LibreTranslate)

→ Run `/grill-me` before implementing (API choice, runtime vs background).

---

## Phase 4 — V1.1 (post-launch)
- [ ] PWA + browser push notifications
- [ ] Monthly report export (PDF)
- [ ] UX adjustments from real feedback
- [ ] **Configurable lead-temperature rules.** Today `DealTemperature` (Hot/Warm/Cold) is fully
      automatic and hardcoded in `ClientService.RecalcTemperature` (≤72h since last interaction =
      Hot, ≤240h = Warm, else Cold) — recalculated only on `UpdateStageAsync`, which sets
      `LastInteractionAt = UtcNow` in the same call, so a stage change always snaps the client back
      to Hot. There is no manual override anywhere — a salesperson who personally judges a lead as
      not promising has no way to mark it Cold themselves. Needs: a settings screen (per-user or
      per-company thresholds) backed by a new rules table, plus a manual override field that wins
      over the automatic calculation until the next stage change.
- [ ] **Recurring notification templates.** Fully designed (data model, generation strategy,
      cancellation rules, test plan) in [[NOTIFICATIONS]], not implemented. Needs a generation
      strategy decision first (lazy on-demand vs a real background worker — this codebase has
      none today) before coding.

## Phase 5 — V1.2+ (future)
- [ ] Multi-stand / groups (manager dashboard per stand)
- [ ] **Affiliate/invite links + join-requests + stand membership management** (2026-06-26 idea,
      not yet designed in detail — replaces the old "Salesperson invite by email" bullet with a
      richer flow). Three pieces, each useful alone:
  1. **Personal affiliate link.** Every user gets a unique invite link (e.g.
     `/register?ref={userId-or-short-code}`) that pre-fills/locks the Company+Brand on the
     registration form. Lets a salesperson recruit colleagues straight into their own stand
     without a Manager doing it manually. Worth tracking *who invited whom* (a `ReferredByUserId`
     on `User`) — cheap to add now, opens the door to later "team growth" stats or gamification
     without re-engineering anything.
  2. **Join-by-search at registration.** Today registration either has no Company/Brand or
     creates a brand-new one (Manager flow) — there's no "I already work at an existing stand,
     let me find it" path for a Salesperson. Add a stand picker (search by name) on the
     registration form; picking one doesn't join immediately — it creates a **pending join
     request** that a Manager/stand-admin of that brand approves or rejects. This is the part
     that needs the most care: a join request is effectively asking a stranger's account to be
     let into a company's data, so it must never auto-approve and the requester must not see any
     of that company's data until approved (same care as the friend-request flow — see
     [[FRIENDS]] for the existing pattern of "visible-but-not-yet-connected" state).
  3. **"Manage members" screen.** A Manager-only page per brand: list of members (pending +
     active), approve/reject join requests, see who invited whom (via the affiliate link above),
     and the existing salesperson activate/deactivate actions (already in the API, just not in one
     dedicated place — currently scattered). This is the natural home for "stand admin" as a
     concept, without needing a new role: it's just the existing `Manager` role scoped to their
     own `Brand`, which the data model already supports.

  Open questions before designing this for real: does a join request need company-side
  visibility into *why* (a note from the requester)? Can a user have a pending request to more
  than one stand at once, or does requesting one auto-cancel others? Worth a `/grill-me` pass
  before implementation — this touches registration, auth, and tenant-isolation security all at
  once, the same category of code that had the cross-tenant Admin-panel bug (see [[SECURITY]]).
- [ ] Lead import (Standvirtual, OLX)
- [ ] SMS / WhatsApp (V2.0)
- [ ] **Real analytics Dashboard, separate from the home screen** (2026-06-26 idea). The page at
      `/dashboard` (`DashboardPage.tsx`) was just renamed "Ecrã Inicial"/"Home" — it's a quick
      daily-glance widget board (this-month KPIs + hot deals + pinned goals), not analytics. The
      plan is a **new, separate page** (`/analytics` or similar) that's genuinely powerful:
  - **Time-range switcher** (Week / Month / Quarter / Year / custom range) that re-drives every
    chart on the page from one control, instead of each KPI being hardcoded to "this month" like
    today's `salesThisMonth`/`proposalsThisMonth`.
  - **Trend charts**, not just single numbers: sales value and count over time (line/bar), funnel
    conversion rate per stage (where do deals actually die?), proposals → sales conversion over
    the selected range.
  - **Stage funnel visualization** — a literal funnel chart showing how many clients are sitting
    in each stage right now, so a Manager spots a bottleneck stage at a glance (e.g. 40 clients
    stuck in "Aguarda Decisão").
  - **Leaderboard for Managers** — rank salespeople in the brand by sales/conversion/value this
    period; ties naturally into the existing Friends comparison feature but scoped to one's own
    team instead of cross-dealership.
  - **Per-brand/vehicle breakdown** — which models/brands are actually selling, for stands with
    more than one represented brand.
  - Today's home-screen widgets (KPI cards, hot deals, pinned goals) should become **clickable**
    deep-links into this new analytics page pre-filtered to the relevant slice (e.g. clicking
    "Vendas este Mês" on the home screen opens the analytics page already set to Month + Sales) —
    a sharper version of the plain page-navigation wired into the home screen on 2026-06-26.
  - Needs new backend aggregation endpoints (today's `DashboardDto` is a fixed shape for "this
    month" only) — per the current data-layer rule, build these as C#/EF Core LINQ in the
    repository, same as `SaleRepository.GetDashboardAsync` (see [[2026-06-29-data-layer-migrated-to-csharp]]), not a SQL function.
- [ ] **Configurable home-screen widgets** (2026-06-27 idea). Today's "Ecrã Inicial" KPI cards are
      a fixed, hardcoded set for every user. The ask: let each salesperson pick which widgets they
      see, up to a cap.
  - **Default: 5 widgets per user, max 5 visible at once** — a user can swap which 5 out of a
    larger catalog are shown, not add unlimited ones (keeps the home screen scannable).
  - **New widget ideas to add to the catalog** (beyond the existing Clientes Ativos/Propostas/
    Vendas/Conversão/Comissão): **Carros Vendidos Este Mês** (count, distinct from the
    revenue-value widget already there), **Carros Entregues** and **Carros Por Entregar**
    (counts from the new `Sale.DeliveredAt`/`EstimatedDeliveryDate` fields — see the Vendas page's
    delivery-status filter, same underlying data), **Objetivos a expirar em breve** (goals within
    X days of their period end and still under target — a different cut than the existing pinned
    goals widget), **Próxima notificação por vencer** (single most-urgent pending notification,
    a "what should I do right now" widget).
  - Needs: a per-user widget preference (which widget keys + their order, capped at 5 — likely a
    `UserHomeWidget` join table or a simple ordered string array on a per-user settings row,
    whichever is cheaper) and a settings UI (probably a "Personalizar" button opening a modal with
    a checklist + the 5-max validation, rather than full drag-and-drop — drag-and-drop reordering
    can be a later refinement, not needed for v1).
  - Decide whether the catalog is hardcoded (a fixed enum of widget types, simplest) or
    eventually data-driven (more flexible, way more work) — start hardcoded, this is exactly the
    kind of decision that should default to the simpler option until proven insufficient.

---

## Milestones

| Milestone | Criterion | Phases |
|-----------|-----------|--------|
| Frontend usable | All pages work, no known bugs | 0 |
| MVP complete | Email verification + audited features | 0–1 |
| Launchable | Payments working | 0–2 |
| Validated | 3–5 real salespeople using it | 0–4 |

---

## Related
[[STATUS]] · [[FRONTEND-ISSUES]] · [[PAYMENTS]] · [[I18N]] · [[OnTime]]
