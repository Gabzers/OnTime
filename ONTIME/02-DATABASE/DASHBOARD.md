# Dashboard

Hub: [[OnTime]] · API: [[API-REFERENCE]] · Schema: [[SCHEMA-PIPELINE]].

The PRD's most important screen. Target load <500ms. `GET /api/dashboard` → `DashboardDto`.

## DashboardDto (verified)
```
ActiveClients              int
ProposalsThisMonth         int
SalesThisMonth             int
RevenueThisMonth           decimal
ConversionRate             decimal
CommissionThisMonth        decimal
MonthlySales               MonthlyStatDto[]   (Year, Month, Proposals, Sales, Revenue)
LossReasons                LossReasonStatDto[] (LossReason, Count)
HotDeals                   object[]
OverdueNotificationsCount  int
```

## Backing stored functions
| Function | Feeds |
|----------|-------|
| `fn_get_dashboard_kpis` | ActiveClients, Proposals/Sales/Revenue/Commission this month, overdue count |
| `fn_get_hot_deals` | HotDeals (temperature=Hot, non-final stage) |
| `fn_get_monthly_stats` | MonthlySales (last N months, proposals vs sales vs revenue) |
| `fn_get_loss_reasons` | LossReasons (grouped count of lost proposals) |

Notes:
- "This month" uses `DATE_TRUNC('month', NOW() AT TIME ZONE 'UTC')`.
- KPIs are scoped to the JWT user. Managers can view a salesperson's dashboard via `GET /api/users/{id}/dashboard` (🔒).
- ConversionRate = sales / proposals × 100 (computed in service/SQL and already returned as percentage points; frontend should only append `%`).

## Frontend
`DashboardPage.tsx` (lazy). Shows today + overdue notifications (see [[NOTIFICATIONS]]), KPI cards, monthly chart, loss-reason chart, hot/warm/cold lists. Charts via `@ant-design/charts`.

## Goal progress
`UserGoalService` reuses this dashboard data to compute live goal progress. See [[GOALS-PERMISSIONS]].

## Related
[[NOTIFICATIONS]] · [[GOALS-PERMISSIONS]] · [[API-REFERENCE]] · [[OnTime]]
