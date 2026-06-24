# Feature: Friends (social network)

Hub: [[OnTimeCRM]] · Schema: [[SCHEMA-AUTH]] · API: [[API-REFERENCE]].

**Implemented** (controller + `FriendshipService` + `UserFriendship`/`UserPublicProfile`). Status: [[STATUS]].

## What it does
Salespeople connect and view each other's public profiles and visible KPIs. Motivation: peer comparison.

## Critical rule
**`Commission` is private — never returned in friend-facing endpoints.** Confirmed by the `SaleDto.Commission` comment. See [[DOMAIN]].

## Entities
- **UserFriendship** — `SenderId`, `ReceiverId`, `Status` (Pending=0, Accepted=1, Rejected=2, Blocked=3). Bidirectional — handle sender/receiver when listing.
- **UserPublicProfile** — visibility settings + aggregated stats, not raw client data.

## Endpoints
```
GET    /api/friends                       accepted friends
GET    /api/friends/requests              pending requests received
POST   /api/friends/requests              send by email { email }
PATCH  /api/friends/requests/{id}/accept
PATCH  /api/friends/requests/{id}/reject
DELETE /api/friends/{friendUserId}        remove
GET    /api/friends/{friendUserId}/profile   public profile + visible KPIs
GET    /api/friends/profile/settings      my visibility settings
PUT    /api/friends/profile/settings      update visibility
```
Also on `/api/users`: `GET/PUT /api/users/me/public-profile` (same service).

## Frontend
`FriendsPage.tsx`, `FriendProfilePage.tsx` exist (lazy-loaded). Verify wiring against the endpoints above.

## Related
[[SCHEMA-AUTH]] · [[DOMAIN]] · [[GOALS-PERMISSIONS]] · [[OnTimeCRM]]
