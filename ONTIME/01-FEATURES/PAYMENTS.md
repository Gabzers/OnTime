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

## Security & PCI scope — non-negotiable before Phase 2 ships

**Never let the OnTime backend touch a raw card number, CVV, or expiry date.** Both providers
support (and Stripe *requires*) tokenization: the card form is Stripe Elements/Ifthenpay's own
hosted widget, running client-side, which returns a token/PaymentMethod ID — that's the only
payment credential that ever reaches our API. This keeps OnTime **out of PCI-DSS SAQ-D scope**
(the expensive, audited tier) and into the much lighter SAQ-A (you never see the card data at
all). If an implementation PR ever adds a raw `CardNumber`/`Cvv` field to a DTO or entity, that's
an immediate security regression — reject it.

**What must be encrypted/protected once Stripe/Ifthenpay land:**
- **Stripe secret key, webhook signing secret, Ifthenpay API keys** — same rule as today's
  `Jwt:Key`/`Brevo:ApiKey`: never in `appsettings.json`, only via Cloud Run env vars, never
  committed. See [[SECURITY]] "Secrets".
- **Webhook signature verification is mandatory, not optional** — both Stripe
  (`Stripe.EventUtility.ConstructEvent` with the raw request body + signing secret) and Ifthenpay's
  callback must be verified before trusting the payload and activating a subscription. An
  unverified webhook endpoint is an open door to "mark my own subscription as Paid" for free.
- **`SubscriptionPayment` rows may store the provider's transaction/reference ID and amount, never
  full card data** — a token/last-4-digits display field (e.g. Stripe's `card.last4`) is fine and
  useful for the user's payment history UI; nothing else card-shaped belongs in our DB.
- **Webhook endpoints must be idempotent** — a provider retrying the same webhook (network hiccup,
  timeout on our side) must not double-activate/double-charge-side-effect a subscription. Key on
  the provider's own event/transaction ID, not just "did this arrive."
- **TLS everywhere** — already true (Cloud Run terminates TLS, Vercel/Supabase both HTTPS-only by
  default) — just confirm no HTTP fallback is ever configured once real payment webhooks are wired.

Run `/grill-me` before starting Phase 2 implementation to lock down the exact Stripe/Ifthenpay
integration shape — this section is the security constraints that any resulting plan must satisfy,
not a full design.

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
