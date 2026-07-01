# Stage-driven temperature, recurring notifications, email delivery, and the pg_cron engine

Date: 2026-06-30 · [[OnTime]] · scoped via `/grill-me`, full transcript not reproduced here — this is the binding spec.

**Status: Sprint 1, in progress.** This supersedes the "lazy/on-demand generation" recommendation
in [[NOTIFICATIONS]]'s "Future: recurring templates" section — the user now wants a real `pg_cron`
job (see §4), so build the recurring-template engine against that instead of the read-path-piggyback
approach originally sketched there. Everything else in that section (data model, stop conditions,
test plan) still applies as written.

## 1. Stage-driven temperature

Today: `Client.Temperature` is derived purely from `LastInteractionAt` (Hot ≤3d, Warm 4–10d, Cold
>10d), recalculated on stage change in `ClientService.RecalcTemperature`. **Problem:** this ignores
*why* the client is at that point in the pipeline — a client with an active proposal is colder by
this rule than one who just walked into the stand, which is backwards.

**New rule:** `ClientStage` gains an optional temperature program:
- `AffectsTemperature` (bool, default false) — opt-in per stage, like the notification opt-in below.
- On entering the stage (same trigger point as `ClientService.UpdateStageAsync` today): if
  `AffectsTemperature`, set `Client.Temperature` to the stage's configured initial value
  immediately (replaces today's hardcoded "non-final stage → always Hot").
- **Time-based transitions**, unlimited per stage, e.g. "Hot immediately → Warm after 7 days → Cold
  after 30 days". Modeled as an ordered list per stage:
  `ClientStageTemperatureRule { ClientStageId, DaysAfterEntry (int, 0 = immediate), Temperature (int) }`,
  ordered by `DaysAfterEntry` ascending. The client's effective temperature at any moment = the
  rule with the largest `DaysAfterEntry` not exceeding `(NOW - entered-this-stage-at)`.
- Stages with `AffectsTemperature = false` keep today's pure last-interaction-based calculation
  (so the feature is additive, not a breaking change for stages that don't opt in).
- `entered-this-stage-at` = the timestamp of the most recent `ClientStageHistory` row for this
  client (already exists, no new column).
- Computed by the **pg_cron job** (§4), not lazily on read — the job walks active stage-temperature
  programs and updates `Client.Temperature` directly, so it's already correct when anyone reads it.

## 2. Notification config moves into the Etapas screen

`StageNotificationTemplate` (entity, endpoints, generation logic) is **unchanged** — only its
**editing surface** moves. `NotificationSettingsPage.tsx` is retired; its table becomes a
drawer/detail view opened from a row action on the Etapas page (`StagesPage`/whatever it's
currently called).

- Each stage row in the Etapas table gains two badge/icon indicators: "tem temperatura configurada"
  (if `AffectsTemperature`) and "tem notificação configurada" (if ≥1 `StageNotificationTemplate`
  row, active or not). Click → opens a drawer with two sections: Temperature rules (CRUD list per
  §1) and Notification templates (CRUD list, same fields as today's table: Título, Dias Após, Hora,
  Recorrência (§3), Ativo).
- No backend route changes needed beyond what §1/§3 add — `StageNotificationTemplate`'s existing
  CRUD endpoints are reused as-is, just called from a different page.

## 3. Recurring notifications — build now (was deferred, pulled into Sprint 1)

Full design already exists in [[NOTIFICATIONS]] "Future: recurring templates" — reuse verbatim:
`IsRecurring`, `RecurrenceIntervalDays`, `FixedDayOfWeek`, `FixedDayOfMonth`, `MaxOccurrences` on
`StageNotificationTemplate`; series tracked via a new `ClientStageNotificationSeries` keyed off
`ClientStageHistory.Id` (fresh series on every stage re-entry, since each entry is a new history
row). Stop conditions confirmed in this session: **stage change** (unconditional) or **Won/Lost**
— matches "the client changes stage → stops unconditionally" already in the original design (Won/
Lost is itself a stage change to a final stage, so no special-case needed). No occurrence cap
requested this session — `MaxOccurrences` stays optional/nullable, defaults to unbounded.

**Only change from the original design:** generation happens in the **pg_cron job** (§4), not
lazily on notification-read — since the job already needs to exist for §1, piggyback both pieces
of logic onto it instead of building two different generation strategies.

## 4. The pg_cron engine — new infra, blocking for Sprint 1 deploy

This codebase has zero background workers today (`BackgroundService`/`IHostedService`/cron — all
absent, confirmed by grep). The user explicitly wants this built *before* Sprint 1 deploys, since
both §1 and §3 depend on something running unattended.

- **Mechanism:** Supabase `pg_cron` extension — schedules a SQL job directly in Postgres (e.g.
  every 5–15 minutes; exact interval TBD at implementation time based on how time-sensitive
  `DaysAfterEntry`/recurrence granularity needs to be — daily-precision rules don't need 5-minute
  polling).
- **What it needs to trigger:** since the actual logic (temperature transitions, notification
  generation, recurrence-series bookkeeping) lives in C#/EF Core per [[2026-06-29-data-layer-migrated-to-csharp]]
  (no SQL business logic), `pg_cron` cannot run this in-database — it needs to call out to the API.
  Two standard patterns: (a) `pg_cron` calls a Postgres `pg_net` HTTP extension to hit a new
  internal `POST /api/internal/run-scheduled-jobs` endpoint (secret-key-guarded, not user-facing),
  or (b) `pg_cron` writes a row to a lightweight "due work" table and the API polls it. **(a) is
  simpler and avoids a polling loop** — confirm `pg_net` is available on Supabase's free tier before
  committing (it's a standard Supabase extension, expected to be, but verify at implementation
  time). This single endpoint becomes the one place that runs: temperature-rule transitions (§1),
  due notification generation including recurrence (§3), and is the future home of the daily digest
  (still not built, but no longer blocked once this exists).
- **Efficiency requirement** (explicit ask): don't do a wide table scan every tick. Scope each run
  to clients/series actually due — e.g. only `ClientStage` rows with `AffectsTemperature` or an
  active recurring series, joined against `ClientStageHistory.CreatedAt` to compute due-ness
  server-side in the query, not by loading everything into C# and filtering in memory.

## 5. Email delivery — Brevo

New C# integration: `IEmailSender`/`BrevoEmailSender`, called via `HttpClient` (Brevo's transactional
email REST API — no SDK dependency needed), triggered whenever a notification fires and the
recipient has opted in (see toggles below). Free tier: 300 emails/day. Chosen over Resend (100/day)
because Brevo also offers SMS on the same platform, which is **the recommendation** for when SMS
work starts in Sprint 2/3 (client-facing email/SMS — explicitly deferred, see §6) — one integration
instead of two.

**Two separate opt-in toggles on `NotificationPreference`** (not one global switch):
- `EmailOnFriendRequests` (bool) — friend-request notifications by email.
- `EmailOnGeneralNotifications` (bool) — everything else (stage notifications, recurring
  occurrences, future digest) by email.

**Registration UX:** both toggles shown with a hover/info icon explaining "cada notificação pode
ir por email além de aparecer na app" — applies at signup and is editable later from Profile (same
pattern as existing `NotificationPreference` fields today). Also add a one-line mention of email
delivery to the app's "Sobre"/About screen.

## 6. Explicitly out of scope this round (Sprint 2/3 — confirmed twice in this session)

- Email verification flow at registration (already tracked, unchanged).
- Automatic email/SMS **to the client** (not the salesperson) + a dedicated screen to view sent/
  received email/SMS history, filterable by client/status/etc. This is a different recipient and a
  different data shape (needs a `ClientCommunicationLog` or similar, inbound+outbound) — deliberately
  not bundled into this round's `NotificationPreference`-based toggles, which are for the
  salesperson's own notifications only.

Affects: [[NOTIFICATIONS]] · [[DOMAIN]] · [[ARCHITECTURE]] · [[ROADMAP]] · [[2026-06-29-data-layer-migrated-to-csharp]]
