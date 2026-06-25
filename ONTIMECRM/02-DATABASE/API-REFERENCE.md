# API Reference

Hub: [[OnTimeCRM]] · Verified against controllers on 2026-06-25.

All routes require JWT `[Authorize]` unless marked. 🔒 = `ManagerOnly` (role 1 or 2, scoped to the
caller's own company). 🔐 = `AdminOnly` (role 2 only, cross-tenant — reads/writes ALL companies).
Confusing the two was a real cross-tenant data leak (fixed 2026-06-25) — see [[SECURITY]].

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
POST /api/proposals/{id}/convert          convert to sale (ConvertToSaleRequest; SoldAt + FinalValue required, Commission optional)
```
Vehicle on the created sale: explicit `ModelId`/`FreeTextModel` in the request wins; otherwise
inferred from the proposal's preferred (or first) `ProposalVehicle`. `SALE_MISSING_VEHICLE` (422)
only if neither is available.

## Sales — `/api`
```
GET /api/sales            paginated ?year&month
GET /api/sales/{id}
PUT /api/sales/{id}       update sale details (finalValue, commission, soldAt, vehicle fields)
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
GET    /                                   all (with templates + clientCount)
POST   /                                   create
PUT    /{id}                               UpdateStageRequest { name, color, isActive, isFinal?, isWon?, isLost? }
DELETE /{id}
POST   /reorder                            ReorderStagesRequest
POST   /{stageId}/templates                add template
PUT    /{stageId}/templates/{templateId}
DELETE /{stageId}/templates/{templateId}
```
Only one stage per user can have `IsWon=true` and one `IsLost=true` — setting either via PUT
auto-demotes the previous holder; `STAGE_WON_AND_LOST` (422) if both requested on the same stage.
`ClientCount` on the list response is live (active clients currently in that stage).

## Vehicles — `/api/vehicles`  (writes 🔒)
```
GET    /brands                              GET open; POST/DELETE 🔒
POST   /brands 🔒   DELETE /brands/{id} 🔒
GET    /models   ?brandId&search&page       — brandId omitted: defaults to caller's selected brands (see USER-BRANDS)
GET    /models/{id}                         lookup a single model by id (used by the proposal editor)
POST   /models 🔒   PUT /models/{id} 🔒   DELETE /models/{id} 🔒
PATCH  /models/{id}/active 🔒               — body { isActive }
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
GET / · GET /requests · GET /search?q=
POST /requests {email} or {userId}
PATCH /requests/{id}/accept · PATCH /requests/{id}/reject
DELETE /{friendUserId} · GET /{friendUserId}/profile
GET|PUT /profile/settings
```
See [[FRIENDS]].

## Users — `/api/users`
```
GET|PUT /me · GET|PUT /me/public-profile · GET|PUT /me/vehicle-brands
GET / 🔒 (brand) · GET /{id} 🔒 · POST / 🔒 (invite)
PUT /{id}/active 🔒 · GET /{id}/dashboard 🔒
```
`PUT /me/vehicle-brands` body `{ brandIds: Guid[] }` replaces the full set. See [[USER-BRANDS]].

## Subscription — `/api/subscription`  (Initiate is stubbed → PAYMENT_PENDING)
```
GET / · GET /payments · POST /initiate · GET /payments/{id}/status · POST /cancel
```
See [[PAYMENTS]].

## Permissions — `/api/permissions` 🔒
```
GET /?role={int} · PUT /{role}   (UpdateMenuPermissionRequest[])
```

## Admin — `/api/admin` 🔐 (AdminOnly — role 2, platform admin, NOT plain ManagerOnly)
```
GET|POST /companies · PUT /companies/{id} · PATCH /companies/{id}/active
GET|POST /companies/{companyId}/brands · PUT|PATCH .../brands/{id}[/active]
```
Cross-tenant by design — lists/edits/disables EVERY company, not just the caller's own. A
Manager (role 1) must get 403 here. Brands for the caller's OWN company are at `/api/brands` 🔒
(CRUD + PATCH /{id}/active) — a different, single-tenant controller.

## Error Logs — `/api/admin/error-logs` 🔐 (AdminOnly)
```
GET /?page&pageSize&statusCode    paginated, most recent first
```
Every error response the API has ever returned (`ErrorLog` entity, written by
`ErrorHandlingMiddleware` for every 4xx/409/500). See [[SECURITY]].

## i18n — `/api/i18n` (anonymous)
```
GET /?locale=pt-PT     hardcoded dictionary, pt-PT and en-US both complete. See [[I18N]].
```

## Related
[[ARCHITECTURE]] · [[NOTIFICATIONS]] · [[DASHBOARD]] · [[OnTimeCRM]]
