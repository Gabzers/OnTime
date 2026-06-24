# Testing

Hub: [[OnTimeCRM]] · Source: `OnTimeCRM.Tests` + `ONTIMECRM_TESTING_SPEC.md`.

## Philosophy
**Integration tests only — no isolated unit tests.** Each test exercises a real end-to-end flow: HTTP → service → real PostgreSQL → back. If it passes, the flow works.

## Stack
| Tool | Role |
|------|------|
| xUnit | test runner |
| Shouldly | readable assertions |
| Microsoft.AspNetCore.Mvc.Testing | in-memory API host |
| Testcontainers.PostgreSql | real Postgres (postgres:16-alpine) |
| Respawn | fast data reset between tests (keeps schema) |
| Bogus | realistic fake data |
| WireMock.Net | mock Stripe + Ifthenpay |

## Infrastructure
- **`TestWebAppFactory`** (`WebApplicationFactory<Program>`, `IAsyncLifetime`): boots API + Postgres container + WireMock; overrides config (connection string, Stripe/Ifthenpay base URLs → WireMock, `Subscription:TrialDays=0`, `GracePeriodDays=0`). Exposes `Client` (HttpClient) and `Db` (AppDbContext).
- **`ResetDatabaseAsync()`**: Respawn cleans modified tables (called in each test class's `InitializeAsync`).
- **`TestHelpers`**: `RegisterManagerAsync`, `RegisterSalespersonAsync`, `LoginAsync`, `ActivateSubscriptionDirectAsync` (bypass payment), `CreateClientWithProposalAsync`. Plus a `PatchAsJsonAsync` extension.
- **`ExternalApiMocks`**: WireMock stubs — Stripe (`/v1/customers`, `/v1/payment_intents[/*][/confirm]`), Ifthenpay (`/mbway/set`, `/mbway/getautoMBWayStatus`, `/multibanco/setreference`).

## Test flows (`OnTimeCRM.Tests/Flows`)
| File | Covers |
|------|--------|
| `ProposalSaleFlowTests` | proposal→sale, **SoldAt preservation**, multiple sales, already-closed 422 |
| `SubscriptionFlowTests` | 402/403 enforcement per AccountStatus, direct activation, public-route bypass |
| `NotificationFlowTests` | auto-notification on stage change, manual, today/overdue, done/snooze, post-sale 30d, prefs |
| `StageConfigFlowTests` | stage + template CRUD |
| `I18nFlowTests` | `/api/i18n` |
| `SecurityAndEdgeCaseTests` | cross-user isolation, edge cases |

## Test pattern
```csharp
// ARRANGE — via TestHelpers
var auth = await TestHelpers.RegisterManagerAsync(_factory.Client);
await TestHelpers.ActivateSubscriptionDirectAsync(_factory.Db, auth.UserId);
// ACT — HTTP call
var resp = await _factory.Client.PostAsJsonAsync("/api/...", req, auth.Token);
// ASSERT — HTTP + direct DB read
resp.StatusCode.ShouldBe(HttpStatusCode.OK);
(await _factory.Db.Sales.FindAsync(id))!.SoldAt.Date.ShouldBe(expected.Date);
```
`_factory.Db.ChangeTracker.Clear()` before re-reading entities mutated via the API.

## Run
```bash
cd OnTimeCRM_Backend && dotnet test    # Docker required (Testcontainers)
```

## Related
[[PAYMENTS]] · [[NOTIFICATIONS]] · [[ARCHITECTURE]] · [[OnTimeCRM]]
