# OnTime — Domain Model

Hub: [[OnTime]] · Schema: [[SCHEMA-PIPELINE]] · [[SCHEMA-AUTH]] · Architecture: [[ARCHITECTURE]].

> Verified against `Domain/Enums/*.cs` and `Domain/Entities/*.cs` on 2026-05-30.

## Hierarchy
```
Company → Brand → User (Salesperson | Manager | Admin)
                    └── ClientStage (7 personal stages, created on registration)
                    └── Client → Proposal → Sale
                    └── Notification
```
- Salesperson sees only own clients. Manager sees all clients in their brand.
- **Admin (role=2) bypasses subscription AND permission checks** (platform superadmin).
- User can register without Company/Brand (both nullable).

---

## Entities (key fields only)

**BaseEntity** — every entity extends it:
`Id(Guid)`, `CreatedAt`, `UpdatedAt` (auto-set in `SaveChangesAsync`), `IsActive(bool)`, `Notes`

**Client** — `UserId`, `LeadSource`, `CurrentStageId`, `Temperature`, `LastInteractionAt`
- Always has ≥1 Proposal (created atomically with the client).
- Temperature derived from `LastInteractionAt`: Hot ≤3d, Warm 4–10d, Cold >10d.

**ClientStage** — per user (not per brand)
- `Order`, `IsFinal`, `IsWon`, `IsLost`
- Move to non-final stage → client `Temperature=0` (Hot). Move to final → temperature unchanged.

**ClientStageHistory** — `FromStageId?`, `ToStageId`, `Obs`, `ProposalSnapshot` (JSON of the active proposal at change time; powers the timeline with no extra queries).

**Proposal** — `Status`, `BusinessType`, `PaymentType`, `ProposalValue`, trade-in fields
- `ProposalDate`: user business date; defaults to UtcNow in service if null.
- Only `Active` proposals can be converted or marked lost.

**Sale**
- `SoldAt`: **NEVER auto-set to UtcNow — always from the request** (dedicated tests enforce this).
- `Commission`: private — never returned in friend-facing endpoints (`SaleDto` comment confirms).

**Notification** — `Trigger`, `Status`, `ScheduledFor`, `DoneAt?`, `SnoozedUntil?`
- `/today` returns Pending where `ScheduledFor ≤ NOW()` (i.e. includes overdue). See [[NOTIFICATIONS]].

**UserGoal** — personal targets. `MetricType`, `Period`, `TargetValue`, `StartDate`, `EndDate?`. Progress computed live from dashboard KPIs. See [[GOALS-PERMISSIONS]].

**MenuItemPermission** — UI-only `(Role, RouteKey)` with `CanView/CanCreate/CanEdit/CanDelete`. API auth (`[ManagerOnly]`/`[Authorize]`) is enforced server-side regardless. See [[GOALS-PERMISSIONS]].

**UserFriendship** — `SenderId`, `ReceiverId`, `Status`. See [[FRIENDS]].

---

## Canonical Enum Reference (serialized as integers)

> Single source of truth. If code changes, update here first.

```
UserRole:            Salesperson=0  Manager=1  Admin=2   (Admin bypasses subscription+permissions)
UserAccountStatus:   PendingActivation=0  Active=1  Expired=2  Inactive=3  Suspended=4  Cancelled=5
SubscriptionStatus:  Trial=0  Active=1  PastDue=2  Cancelled=3  Expired=4
SubscriptionPlan:    Trial=0  Monthly=1  Annual=2
ProposalStatus:      Active=0  Won=1  Lost=2  Cancelled=3
BusinessType:        DirectPurchase=0  TradeIn=1  TradeInWithDifference=2  Leasing=3  Financing=4
PaymentType:         Cash=0  Financing=1  Leasing=2  BankTransfer=3  Other=4
DealTemperature:     Hot=0  Warm=1  Cold=2
LeadSource:          no longer an enum (2026-06-29) — `Client.LeadSource` (int) now references the calling
                     company's `LeadSourceOption.Code`. New companies are seeded with these 8 defaults
                     (same codes 0-7: Stand/Telefone/OLX/Standvirtual/Instagram/Facebook/Referência/Outro)
                     but Manager/Admin can rename/add/deactivate via `/api/lead-sources` (LeadSourcesPage).
                     See [[ROADMAP]].
LossReason:          Price=0  Competition=1  NoDecision=2  FinancingRejected=3  VehicleNotAvailable=4  ClientUnreachable=5  Other=6
NotificationStatus:  Pending=0  Done=1  Snoozed=2  Ignored=3
NotificationTrigger: Manual=0  StageChanged=1  SaleClosed=2  ProposalCreated=3  Custom=4
FriendshipStatus:    Pending=0  Accepted=1  Rejected=2  Blocked=3
GoalMetricType:      NewClients=0  Sales=1  Proposals=2  ConversionRate=3
GoalPeriod:          Daily=0  Weekly=1  Monthly=2
FuelType:            Petrol=0  Diesel=1  Electric=2  Hybrid=3  PluginHybrid=4  LPG=5  Other=6
TradeInType:         Car=0  Motorcycle=1  Van=2  Truck=3  Other=4
PaymentProvider:     Stripe=0  Ifthenpay=1
PaymentMethodType:   Card=0  MBWay=1  Multibanco=2
SubscriptionPaymentStatus: Pending=0  Paid=1  Failed=2  Refunded=3  Expired=4
```

> ⚠️ The i18n labels for `NotificationStatus` are swapped (2↔3) — see [[FRONTEND-ISSUES]].

---

## Related
[[NOTIFICATIONS]] · [[GOALS-PERMISSIONS]] · [[FRIENDS]] · [[SCHEMA-PIPELINE]] · [[CONVENTIONS]] · [[OnTime]]
