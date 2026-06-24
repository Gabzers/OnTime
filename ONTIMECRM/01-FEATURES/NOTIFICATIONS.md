# Feature: Notifications ⭐

Hub: [[OnTimeCRM]] · Schema: [[SCHEMA-PIPELINE]] · Domain: [[DOMAIN]] · API: [[API-REFERENCE]].

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

## Related
[[SCHEMA-PIPELINE]] · [[DOMAIN]] · [[DASHBOARD]] · [[FRONTEND-ISSUES]] · [[OnTimeCRM]]
