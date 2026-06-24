# Centralized access/scope helper

Date: 2026-05-30 · Status: **implemented 2026-06-06** · [[OnTimeCRM]]

**Rule (proposed):** all controllers/services derive ownership scope through one helper, not
ad-hoc claim reading. Admin-bypass is encoded in the helper, not duplicated per controller.

**Why:** the current ad-hoc pattern produced two real bugs:
- `ClientsController` uses `User.IsManager() ? GetBrandId() : null` — `IsManager()` returns `role == 1`, so **Admin (role=2) gets the salesperson view** of clients instead of brand-wide. The `ManagerOnly` policy correctly allows 1 OR 2; the helper does not. Asymmetric.
- 9 sites use `User.TryGetCompanyId() ?? Guid.Empty` (or BrandId) as a fallback. When the claim is missing, this **silently queries with an empty Guid → zero results, no error**.

**Implies (do / don't):**
- ✅ Add `IsAdmin()`, `IsManager()` (kept as role==1), `IsManagerOrAdmin()` to `ClaimsPrincipalExtensions`.
- ✅ Introduce a small `AccessScope` value type built from `ClaimsPrincipal`:
  ```csharp
  public readonly record struct AccessScope(Guid UserId, Guid? BrandId, Guid? CompanyId, int Role)
  {
      public bool IsAdmin              => Role == 2;
      public bool IsManagerOrAdmin     => Role >= 1;
      public Guid? ManagerBrandScope   => IsManagerOrAdmin ? BrandId : null;   // null = own-only
  }
  // Helper: User.Scope() → AccessScope
  ```
- ✅ Controllers pass `User.Scope()` (or just the parts they need) to services. No more `?? Guid.Empty`.
- ✅ When a claim is genuinely required but missing → throw `AUTH_FORBIDDEN`, never silently fall back to `Guid.Empty`.
- ❌ Don't read claims directly in controllers anymore; the helper is the single source.

**Bugs this fixes (track in [[KNOWN-ISSUES]]):**
- Admin sees salesperson-scoped clients (ClientsController).
- Manager without a brand claim sees empty results instead of an error (Brands/Users controllers).
- Future endpoints can't accidentally re-introduce the same asymmetry.

**Rejected:** keep ad-hoc reads → the inconsistency keeps recurring; bigger blast radius as endpoints grow.

Affects: [[ARCHITECTURE]] · [[CONVENTIONS]] · [[SECURITY]] · [[GOALS-PERMISSIONS]]
