# Feature: Payments & Subscriptions

Hub: [[OnTime]] · Schema: [[SCHEMA-AUTH]] · Roadmap: [[ROADMAP]] (Phase 2).

## Status: stub
`SubscriptionService` is a deliberate stub (class comment: "payment integration is post-MVP"):
- `GetStatusAsync` → returns defaults derived from `User.AccountStatus`
- `GetPaymentsAsync` → empty list
- `InitiateAsync` → **throws `ApiException(PAYMENT_PENDING)`**
- `GetPaymentStatusAsync` → throws `PAYMENT_NOT_FOUND`
- `CancelAsync` → no-op

There is **no Stripe/Ifthenpay client** in Infrastructure yet (only EF columns + WireMock stubs in tests). See [[STATUS]].

## Plans
- Monthly: 15€/user/month
- Annual: 149€/user/year (~12.40€/month, ~17% off)
- Free trial: 14 days, no card

## Methods (target)
| Method | Provider | Region |
|--------|----------|--------|
| Card | Stripe | International |
| MBWay | Ifthenpay | Portugal |
| Multibanco reference | Ifthenpay | Portugal |

Enums: `PaymentProvider` (Stripe=0, Ifthenpay=1), `PaymentMethodType` (Card=0, MBWay=1, Multibanco=2), `SubscriptionPaymentStatus` (Pending=0, Paid=1, Failed=2, Refunded=3, Expired=4). See [[DOMAIN]].

## Implementation plan (Phase 2)
- **Stripe**: create customer → PaymentIntent → return client_secret → webhook `payment_intent.succeeded` activates; `invoice.payment_failed` → PastDue + grace.
- **Ifthenpay MBWay**: POST phone → user approves in app → webhook activates.
- **Ifthenpay Multibanco**: get Entity+Reference+expiry → user pays → callback activates.
- WireMock stubs already model these calls in tests (`ExternalApiMocks.cs`). See [[TESTING]].

## Subscription state machine
```
Register → PendingActivation (trial, TrialEndsAt = now+14d)
PendingActivation + payment → Active
Active → Expired (expiry passed, 3-day grace via GracePeriodDays)
Expired → Suspended (grace elapsed)
Active → Cancelled (user cancels)
Expired/Suspended → Active (renewal)
```

## Endpoints (exist; Initiate currently throws)
```
GET  /api/subscription                     status
GET  /api/subscription/payments
POST /api/subscription/initiate            → PAYMENT_PENDING (stub)
GET  /api/subscription/payments/{id}/status
POST /api/subscription/cancel
```

## Related
[[SCHEMA-AUTH]] · [[ARCHITECTURE]] (subscription middleware) · [[TESTING]] · [[OnTime]]
