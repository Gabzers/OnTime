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
GET    /api/friends/requests/sent         pending requests sent (added 2026-06-25)
GET    /api/friends/search?q=             search active users by name or email (autocomplete)
POST   /api/friends/requests              send by { email } or { userId }
PATCH  /api/friends/requests/{id}/accept
PATCH  /api/friends/requests/{id}/reject
DELETE /api/friends/requests/{id}         cancel a request you sent (added 2026-06-25 — sender only, 403 otherwise)
DELETE /api/friends/{friendUserId}        remove
GET    /api/friends/{friendUserId}/profile   public profile + visible KPIs
GET    /api/friends/profile/settings      my visibility settings
PUT    /api/friends/profile/settings      update visibility
```
Also on `/api/users`: `GET/PUT /api/users/me/public-profile` (same service).

### Search (added 2026-06-24, hardened 2026-06-25)
`GET /api/friends/search?q=` matches `FullName`/`Email` (case-insensitive, partial, max 10
results), excludes the caller, and flags each result with `alreadyFriend`/`requestPending` so the
UI can disable re-sending. Returns `FriendSearchResultDto` (userId, fullName, email, avatarUrl,
brandName, companyName, alreadyFriend, requestPending). `SendFriendRequestDto` accepts either
`Email` or `UserId` — prefer `UserId` when the user was picked from a search result (avoids
ambiguity if two users could share characteristics).

This search is intentionally **not** scoped to company/brand — it's the whole point of the
cross-dealership social feature. That makes it a directory-scraping target, so: query must be
≥2 characters (was: any non-empty string), and `email` in the response is masked
(`j***@example.com`) — never sent in full to someone who isn't a friend yet.

**Email masking extended to pending requests** (2026-06-25) — `GET /requests` and
`GET /requests/sent` used to return the full, unmasked email even though search already masked
it; same `j***@example.com` mask now applied for consistency (the response describing the
sender/receiver's own *self*-info, e.g. the body of `POST /requests`, is still unmasked — that's
the caller's own email, not someone else's).

**Bug fixed 2026-06-25:** `POST /requests`'s response had `SenderEmail` populated with the
RECEIVER's email and an empty `SenderName` — `FriendshipService.SendRequestAsync` built the DTO
with the wrong person's data. No test ever asserted the response body's actual field values
(only the status code), so it went unnoticed. Regression test added.

## Frontend
`FriendsPage.tsx`, `FriendProfilePage.tsx` exist (lazy-loaded).
"Adicionar Amigo" uses a debounced AntD `AutoComplete` against `/api/friends/search`; each option
shows a hover `Popover` with email/brand/company, then a confirmation chip before sending the
request. Tests: `FriendsFlowTests.cs` (search by name/email, self-exclusion, status flags, send by
userId).

## Related
[[SCHEMA-AUTH]] · [[DOMAIN]] · [[GOALS-PERMISSIONS]] · [[OnTimeCRM]]
