# Project review — 2026-05-30

Session log (snapshot). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** critical project-wide review; surface bugs/debt the previous passes missed; answer the
"permission helper?" question; correct the over-cautious "no local AI access" claim.

## Done
- Updated [[AI-WORKFLOW]] to clarify gate ownership (local model writes; gate is executed by you / Claude / Continue agent mode).
- Updated [[LOCAL-AI-SETUP]] with a real "give it more access" section (agent mode + MCP + caveats).
- New decision proposal: [[2026-05-30-access-scope-helper]].
- Added 6 new backend bugs + 6 tech-debt items to [[KNOWN-ISSUES]].

## Findings — what's good
Clean architecture is properly separated; per-entity ownership checks present; constant-time
password compare; secrets gitignored; integration tests on a real Postgres; clean generic
frontend components; brand-color theming; i18n with offline fallback.

## Findings — bugs (now in [[KNOWN-ISSUES]])
1. **Admin gets salesperson view on `/api/clients`** — `IsManager()` is `role==1`; Admin (2) fails it. `ManagerOnly` policy allows both. Asymmetry.
2. **`?? Guid.Empty` claim fallback (9 sites)** — silent zero results instead of 401/403.
3. **`SubscriptionService.GetStatusAsync` wrong shape** — puts `accountStatus` into the subscription status field, `Plan` hardcoded to 1.
4. **`AuthService.LoginAsync` IsActive check inverted** — soft-deleted users with `Status=Inactive` can still log in (then 403 later).
5. **`PermissionService` collapses 4 flags into 2 on update** — `CanCreate`/`CanDelete` are not actually configurable despite existing in the schema.
6. **Trial lockout** (already in T1).

## Findings — tech debt (also in [[KNOWN-ISSUES]])
No CI/CD · no structured logging · no HTTPS redirection · no rate limiting · `AppDbContext`
implements both `IAppDbContext`+`IUnitOfWork` · `ProposalsController` routing anomalous · test
config trivializes trial/grace (masked T1) · schema-drift sentinel column is fragile.

## Permission helper answer
Yes — proposed in [[2026-05-30-access-scope-helper]]. A small `AccessScope` value type derived from
`ClaimsPrincipal` centralizes admin-bypass and removes the `?? Guid.Empty` foot-gun in one shot.
Fixes bugs (1) and (2) above and prevents the next inconsistency.

## Next
- Decide on the access-scope helper (accept → demote `IsManager()`, add `IsAdmin()`/`IsManagerOrAdmin()`/`Scope()`, sweep controllers).
- Slot the new bugs into [[AI-TASK-PLAN]] — bug (1) and (2) become a single TDD task once the helper exists; bugs (3)–(5) are independent.

## Links
[[AI-TASK-PLAN]] · [[KNOWN-ISSUES]] · [[2026-05-30-access-scope-helper]] · [[SECURITY]]
