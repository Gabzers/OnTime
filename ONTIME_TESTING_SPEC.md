# OnTimeCRM Testing Spec

This file is intentionally short. The vault under `ONTIMECRM/` is the source of truth.

## Test stack
- Integration tests only.
- Real HTTP with `WebApplicationFactory<Program>`.
- Real PostgreSQL with Testcontainers.
- `Respawn` to reset DB state between tests.
- `WireMock.Net` for payment webhook mocks.

## Coverage areas
- Auth and onboarding.
- Subscription and payments.
- Client pipeline.
- Proposal and sale flows.
- Notifications.
- Multi-tenancy and access scope.
- Vehicle and user-brand filtering.

## Commands
```bash
dotnet test .\OnTimeCRM_Backend\OnTimeCRM.sln
```

## Rules
- No unit-test-only strategy for business logic.
- Prefer end-to-end coverage for public flows.
- Validate the touched slice after code changes.

## References
- `ONTIMECRM/02-DATABASE/TESTING.md`
- `ONTIMECRM/00-PROJECT/ARCHITECTURE.md`
- `ONTIMECRM/00-PROJECT/CONVENTIONS.md`
- `ONTIMECRM/01-FEATURES/NOTIFICATIONS.md`
