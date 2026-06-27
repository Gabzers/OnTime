# OnTimeCRM Backend Master Prompt

This file is intentionally short. The vault under `ONTIMECRM/` is the source of truth.

## Read first
- `ONTIMECRM/OnTimeCRM.md`
- `ONTIMECRM/00-PROJECT/ARCHITECTURE.md`
- `ONTIMECRM/00-PROJECT/CONVENTIONS.md`
- `ONTIMECRM/00-PROJECT/DOMAIN.md`
- `ONTIMECRM/00-PROJECT/STATUS.md`
- `ONTIMECRM/02-DATABASE/API-REFERENCE.md`
- `ONTIMECRM/02-DATABASE/SCHEMA-PIPELINE.md`
- `ONTIMECRM/02-DATABASE/TESTING.md`
- `ONTIMECRM/04-DECISIONS/2026-05-30-data-layer.md`

## Core rules
- English only for code and docs.
- No EF migrations.
- `SoldAt` always comes from the request.
- `commission` is private.
- Admin role bypasses subscription and permission checks.
- Keep read flows and write flows consistent with the docs.
- Update the matching vault note before closing a session when code changes a documented rule.

## Architecture reminders
- ASP.NET Core 8 + EF Core + PostgreSQL.
- `Presentation/Program.cs` is the composition root.
- Each module exposes `Setup.ConfigureServices`.
- Simple CRUD goes in C# services; complex aggregate reads may use `fn_*` functions.
- DTOs are `record`s; API JSON is camelCase and enums are integers.

## Main backend flows
- Create client + first proposal atomically.
- Update stage in a single transaction and generate notifications.
- Convert proposal to sale without auto-setting `SoldAt`.
- Sales list/detail/dashboard are separate concerns.
- Notification preferences and scheduled notifications must keep their field names in sync.

## Build and test
```bash
dotnet build .\ERP_BACKEND\Presentation\Presentation.csproj
dotnet run --project .\ERP_BACKEND\Presentation\Presentation.csproj
dotnet test .\OnTimeCRM_Backend\OnTimeCRM.sln
```

## Notes
- Do not print or commit secrets.
- Do not add migrations.
- Do not duplicate a flow in SQL and C#.
- Keep files small; split before they become hard to review.
