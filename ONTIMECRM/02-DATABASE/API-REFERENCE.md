# API Reference

Hub: [[OnTimeCRM]] · Verified against controllers on 2026-05-30.

All routes require JWT `[Authorize]` unless marked. Manager-only marked 🔒. Admin (role=2) bypasses checks.

## Auth — `/api/auth` (anonymous)
```
POST /register-manager        company + brand + manager onboarding
POST /register                salesperson under existing brand
POST /login
POST /logout                  (auth) stateless no-op
GET  /companies               public company list (registration)
GET  /companies/{companyId}/brands   public brand list
```

## Clients — `/api/clients`
```
GET    /                 paginated (manager sees brand); ?stageId&temperature&leadSource&search&page&pageSize
POST   /                 create client + first proposal (atomic)
GET    /hot              hot deals
GET    /{id}
DELETE /{id}             soft delete
PUT    /{id}/stage       change stage (core flow)
GET    /{id}/history     stage history
GET    /{id}/sales       sale history
```

## Proposals
```
GET  /api/proposals                      paginated
GET  /api/proposals/{id}
PUT  /api/proposals/{id}                  update (CreateProposalRequest)
POST /api/clients/{clientId}/proposals    create for client
POST /api/proposals/{id}/lost             mark lost (MarkProposalLostRequest)
POST /api/proposals/{id}/convert          convert to sale (ConvertToSaleRequest; SoldAt required)
```

## Sales — `/api`
```
GET /api/sales            paginated ?year&month
GET /api/sales/{id}
GET /api/dashboard        DashboardDto. See [[DASHBOARD]].
```

## Notifications — `/api/notifications`  (actions are PATCH)
```
GET   /                   paginated ?status
GET   /today              ScheduledFor ≤ NOW & Pending (incl. overdue)
GET   /overdue-count
POST  /                   create manual
PATCH /{id}/done
PATCH /{id}/snooze        { snoozedUntil }
PATCH /{id}/ignore
```
Preferences: `GET|PUT /api/preferences/notifications`. See [[NOTIFICATIONS]].

## Stages — `/api/stages`
```
GET    /                                   all (with templates)
POST   /                                   create
PUT    /{id}
DELETE /{id}
POST   /reorder                            ReorderStagesRequest
POST   /{stageId}/templates                add template
PUT    /{stageId}/templates/{templateId}
DELETE /{stageId}/templates/{templateId}
```

## Vehicles — `/api/vehicles`  (writes 🔒)
```
GET    /brands                              GET open; POST/DELETE 🔒
POST   /brands 🔒   DELETE /brands/{id} 🔒
GET    /models   ?brandId&search&page
POST   /models 🔒   PUT /models/{id} 🔒   DELETE /models/{id} 🔒
GET    /models/{modelId}/versions
POST/PUT/DELETE versions 🔒
```

## Goals — `/api/goals`  (scoped to JWT user)
```
GET / · POST / · PUT /{id} · DELETE /{id}
```
See [[GOALS-PERMISSIONS]].

## Friends — `/api/friends`
```
GET / · GET /requests · POST /requests {email}
PATCH /requests/{id}/accept · PATCH /requests/{id}/reject
DELETE /{friendUserId} · GET /{friendUserId}/profile
GET|PUT /profile/settings
```
See [[FRIENDS]].

## Users — `/api/users`
```
GET|PUT /me · GET|PUT /me/public-profile
GET / 🔒 (brand) · GET /{id} 🔒 · POST / 🔒 (invite)
PUT /{id}/active 🔒 · GET /{id}/dashboard 🔒
```

## Subscription — `/api/subscription`  (Initiate is stubbed → PAYMENT_PENDING)
```
GET / · GET /payments · POST /initiate · GET /payments/{id}/status · POST /cancel
```
See [[PAYMENTS]].

## Permissions — `/api/permissions` 🔒
```
GET /?role={int} · PUT /{role}   (UpdateMenuPermissionRequest[])
```

## Admin — `/api/admin` 🔒
```
GET|POST /companies · PUT /companies/{id} · PATCH /companies/{id}/active
GET|POST /companies/{companyId}/brands · PUT|PATCH .../brands/{id}[/active]
```
Brands also at `/api/brands` 🔒 (CRUD + PATCH /{id}/active).

## i18n — `/api/i18n` (anonymous)
```
GET /?locale=pt-PT     hardcoded dictionary. See [[I18N]].
```

## Related
[[ARCHITECTURE]] · [[NOTIFICATIONS]] · [[DASHBOARD]] · [[OnTimeCRM]]
