# Feature: Notifications ⭐

Hub: [[OnTime]] · Schema: [[SCHEMA-PIPELINE]] · Domain: [[DOMAIN]] · API: [[API-REFERENCE]].

The core feature. Everything else supports it.

## What it does
Reminds salespeople to act at the right time:
- Client enters a stage → notification created N days later (from a template)
- Sale closed → post-sale follow-up notification (default 30 days)
- Manual notifications for specific dates

## Lifecycle
`Pending → Done | Snoozed (rescheduled) | Ignored`

## Automatic creation (in C# services + EF, one SaveChanges)

> Runs in the C# services, NOT the same-named stored functions (which are dead). See [[2026-05-30-data-layer]].

**On stage change — `ClientService.UpdateStageAsync`:**
1. Insert `ClientStageHistory` with proposal snapshot (`BuildSnapshot`)
2. Update stage + last_interaction_at; recalc temperature if stage not final
3. For each enabled template on the target stage → Notification at `NOW + DaysAfter` (`GenerateFromTemplates`)

**On sale conversion — `ProposalService.ConvertToSaleAsync`:**
1. Create sale (`SoldAt = req.SoldAt`, never UtcNow) → proposal Won → client to IsWon stage + history
2. Read `NotificationPreference.SaleFollowUpDays` (default 30)
3. Post-sale notification from the Won stage's templates at `NOW + N days`
   Requires a vehicle (`ModelId`/`FreeTextModel`) else `SALE_MISSING_VEHICLE`. Can cancel siblings via `CancelSiblingProposalIds`.

## Default templates (created per new user)
| Stage | Title | Days after |
|-------|-------|------------|
| Visita Agendada | Confirmar visita | 1 |
| Aguarda Decisao | Ligar ao cliente | 2 |
| Venda | Contacto pos-venda | 30 |

## Endpoints (verified — note PATCH, not PUT)
```
GET   /api/notifications                  paginated, filter by status
GET   /api/notifications/today            ScheduledFor ≤ NOW & status=Pending (includes overdue)
GET   /api/notifications/overdue-count
POST  /api/notifications                  create manual
PATCH /api/notifications/{id}/done
PATCH /api/notifications/{id}/snooze      body: { snoozedUntil: DateTimeOffset }
PATCH /api/notifications/{id}/ignore
GET   /api/preferences/notifications
PUT   /api/preferences/notifications
```

> ⚠️ Frontend bug: the snooze call sends `{ snoozeUntil }` but the API expects `snoozedUntil`. See [[FRONTEND-ISSUES]].

## Preferences (per user)
`DailyDigestTime` (09:29), `DigestFrequencyDays` (2), `SaleFollowUpDays` (30), `DigestEnabled`, `StageChangeNotificationsEnabled`, `SaleNotificationsEnabled`, `NewClientNotificationDaysAfter` (2), `NewClientNotificationTime`.

## Dashboard
Shows today + overdue; `fn_get_overdue_count` feeds the bell badge (polled every 60s). See [[DASHBOARD]].

## Future: recurring templates (design only, not implemented)

A stage can already have multiple templates today, and `NotificationSettingsPage.tsx` already
flattens every stage's templates into individual rows — both verified in code, no change needed.

The open ask is a **recurring** mode: instead of firing exactly once per client-stage-entry, keep
firing every N days while the client stays in that stage.

**Semantics (scoped via `/grill-me`, don't re-litigate without re-asking the user):**
1. **Opt-in per template** (`IsRecurring` flag) — existing one-shot templates are unaffected.
2. **Scoped to one stage visit.** The series belongs to this client's current stay in this stage,
   not the template config globally and not the client's lifetime. Re-entering the same stage
   later (e.g. via "New Opportunity" below) starts a fresh series from occurrence 1.
3. **What stops the series:**
   - Marking the *generated* `Notification` **Done** → cancels all future occurrences for this
     stage visit only. The template config stays active for the next client who enters the stage.
   - The client **changes stage** → stops unconditionally, Done or not.
   - An optional **`MaxOccurrences`** cap is reached.
   - Snooze/Ignore do **not** stop the series — only Done does.
4. The occurrence count always resets to zero on stage (re-)entry — never a lifetime counter.
5. **Scheduling needs to support both**, deliberately flexible: an **interval** mode (repeat
   every `RecurrenceIntervalDays` after the existing `DaysAfter` first fire) and a **fixed-day**
   anchor mode (a specific weekday or day-of-month instead of a pure interval).

**Data model:** `StageNotificationTemplate` would gain `IsRecurring` (bool), `RecurrenceIntervalDays`
(int?), `FixedDayOfWeek` (int? 0-6), `FixedDayOfMonth` (int? 1-31 — mutually exclusive with the
other two), `MaxOccurrences` (int?, null = unbounded). Needs a new way to track "how many times
has this template fired during the client's current stage visit" — recommended: a join entity
keyed off `ClientStageHistory.Id` (e.g. `ClientStageNotificationSeries`), since a new stage entry
is already a new `ClientStageHistory` row, so "fresh series on re-entry" falls out naturally.

**Generation strategy:** this codebase has zero background workers/cron today. Recommended MVP
approach is lazy/on-demand generation — when notifications are read (`GetTodayAsync`/
`GetOverdueAsync`), check if the active series is due for its next occurrence and create it then,
piggybacking on existing read paths instead of standing up new scheduler infrastructure. A real
background worker is the "proper" fix but is a new infra category for the project, and Render's
free tier sleeping after 15 min idle complicates an in-process timer anyway.

**"New Opportunity" (re-engaging an already Won/Lost client) — separate ask, already solved:**
no backend guard stops moving a client from a final stage back to an active one, and no
constraint limits proposals/sales per client. The only gap was UI discoverability — fixed with a
"Nova Oportunidade" button on `ClientDetailPage` (visible when `currentStageIsFinal`) that moves
the client to the lowest-`Order` non-final stage in one click.

**Test plan for whoever implements this** (backend, xUnit):
1. Creating a recurring template with both interval and fixed-day set → 422 (mutual exclusion).
2. Client enters a stage with a recurring template → first occurrence at `DaysAfter`.
3. Marking an occurrence Done cancels only *future* occurrences for *this* visit — a different
   client entering the same stage afterward still gets occurrence 1.
4. Client changes stage → series stops regardless of Done state.
5. Series reaches `MaxOccurrences` → stops generating further ones.
6. Client re-enters the same stage after leaving → fresh series from occurrence 1 (covers
   Won → New Opportunity → same stage again).
7. Snooze/Ignore do not cancel the series — only Done does.

**Open questions to ask before building (don't assume):** exact UI shape for the fixed-day anchor
(one "day type" selector with conditional inputs, or two separate optional fields?); whether
`MaxOccurrences` should default to a pre-filled number or start empty/unbounded; whether
`OverridesNewClientNotification` interacts with recurring templates at all or is one-shot-only by
definition (not discussed in the original interview).

## Related
[[SCHEMA-PIPELINE]] · [[DOMAIN]] · [[DASHBOARD]] · [[FRONTEND-ISSUES]] · [[OnTime]]
