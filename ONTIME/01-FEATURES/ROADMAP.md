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

## Phase 1 — Close MVP gaps (NOW)

- [x] **F1 — User Brands filter** ✅ 2026-06-06. See [[USER-BRANDS]].
- [ ] **Email verification** flow (post-registration). See [[SCHEMA-AUTH]].
- [x] Audit the implemented Goals/Friends/Permissions pages end-to-end ✅ 2026-06-25 (full Chrome sweep + test-quality audit — see [[STATUS]]). Found and fixed: Goals date-window bug, Friends email/name-swap bug, the cross-tenant Admin-panel authz hole. See [[GOALS-PERMISSIONS]], [[FRIENDS]], [[SECURITY]].
- [ ] Confirm dashboard performance target (<500ms). See [[DASHBOARD]].
- [ ] Data-layer reconciliation (dead/drifted `fn_*` functions). See [[2026-05-30-data-layer]].

---

## Phase 1.5 — UX & admin polish (2026-06-26 backlog)

User-requested batch, clarified via `/grill-me` before being recorded here — none of these are implemented yet.

- [ ] **Goals page + home screen visual redesign.** Make `/goals` and the pinned-goals widget on the
      home screen more visually engaging/motivating (progress feels like an achievement, not a
      table row). No functional change, pure UX/visual.
- [ ] **Optional "already has a proposal" checkbox on client creation.** Today client creation
      always atomically creates ≥1 `Proposal` (see [[DOMAIN]] — "Always has ≥1 Proposal"). Add a
      checkbox so the salesperson can create a client with **zero** proposals; if unchecked, the
      client has no proposal until one is created later from the Propostas screen. This changes a
      core domain invariant — `ClientService.CreateAsync` and any code assuming `client.Proposals`
      is non-empty needs auditing, not just the frontend form.
- [ ] **Lead Source maintenance screen.** `LeadSource` is currently a fixed backend enum (see
      [[DOMAIN]]) — add a real CRUD screen (like Marcas/Filiais) so it becomes user-configurable
      data instead of a hardcoded list.
- [ ] **"Not an automotive account" profile toggle.** Hides vehicle-related UI (Veículos page,
      vehicle picker in proposals) for non-automotive tenants — **UI-hide only for now**, no data
      model change. Explicitly scoped narrow: if this validates well, a more generic "product"
      model (replacing "vehicle") is a separate future decision, not part of this.
- [ ] **Admin panel: fix broken expanded-row layout.** Expanding a company/brand row in
      `/admin` renders a cramped, overflowing horizontal-scroll table overlapping the row above it
      (see screenshot in session — needs its own card/spacing, not a squeezed inline table).
- [ ] **Profile: let a user edit their own email + personal data.** Currently presumably locked
      after registration — needs a `PUT /api/users/me` (or extend the existing one) covering
      email/name, plus checking whether changing email needs re-verification once email
      verification (Phase 1, still open) exists.
- [ ] **Admin panel: change a user's role; registration always defaults to lowest access.**
      Scope clarified via `/grill-me`: **only the platform Admin** (role=2, cross-tenant) can
      change a user's role (Salesperson/Manager/Admin) — this is not a per-company Manager
      capability, and the existing per-role Access Control screen (see [[GOALS-PERMISSIONS]]) is
      *not* being replaced or extended to per-user overrides. Two parts: (1) confirm/enforce that
      registration always creates a Salesperson (role=0), never anything higher — audit
      `AuthService`/registration endpoints for this; (2) the user's own Profile page should clearly
      show their current role so they always know what kind of account they have. Security note:
      this is the same category of code as the cross-tenant Admin-panel bug fixed 2026-06-25 (see
      [[SECURITY]]) — role changes must stay behind the `AdminOnly` policy, never `ManagerOnly`.
- [ ] **Optional license-plate (matrícula) field per vehicle row.** Add to the vehicle-row level
      (the same `VehicleProposalTable` row used by both proposal creation and convert-to-sale, per
      `/grill-me` clarification — a vehicle may already have a plate if used/pre-registered, or
      none if genuinely new) — optional, free text.

**Also found while documenting (not requested, flagged proactively):**
- [x] **Fixed immediately (2026-06-26):** the new Stages-page notification Popover (bell icon, see
      [[STATUS]] "single-color auto-fill" entry) used `trigger="hover"` only, which never opens on
      touch devices (no hover state) — changed to `trigger={['hover', 'click']}`.

---

## Phase 2 — Monetization

- [ ] **Stripe** — checkout + webhook. See [[PAYMENTS]].
- [ ] **Ifthenpay** — MBWay + Multibanco + callback. See [[PAYMENTS]].
- [ ] Subscription page wired to real `InitiateAsync` (currently throws `PAYMENT_PENDING`).

→ Depends on Phase 0. Unblocks real launch.

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
    month" only) — likely a `fn_*` SQL function per chart given the data-layer split rule for
    aggregate/multi-table logic (see [[2026-05-30-data-layer]]), not more C# service loops.
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
