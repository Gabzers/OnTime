# OnTimeCRM Product Requirements

This file is intentionally short. The vault under `ONTIMECRM/` is the source of truth.

## Product
OnTimeCRM is a CRM SaaS for car-dealership sales teams in Portugal.
The product focus is on pipeline management, follow-up notifications, proposals, sales, vehicles, and multi-tenant access control.

## Roles
- Salesperson
- Manager
- Admin

## Core flows
- Register manager and salesperson accounts.
- Create a client with an initial proposal.
- Change stage and generate follow-up notifications.
- Update proposals and convert them into sales.
- Track sales, commission, dashboard KPIs, and notifications.
- Manage user-selected vehicle brands and brand/model availability.

## Non-negotiables
- English for code and docs.
- UI text from i18n only.
- No EF migrations.
- `SoldAt` must come from the request.
- `commission` stays private.
- Page sizes must stay bounded.

## References
- `ONTIMECRM/OnTimeCRM.md`
- `ONTIMECRM/00-PROJECT/ARCHITECTURE.md`
- `ONTIMECRM/00-PROJECT/DOMAIN.md`
- `ONTIMECRM/01-FEATURES/ROADMAP.md`
- `ONTIMECRM/01-FEATURES/NOTIFICATIONS.md`
- `ONTIMECRM/01-FEATURES/VEHICLES.md`
- `ONTIMECRM/02-DATABASE/API-REFERENCE.md`
- `ONTIMECRM/02-DATABASE/DASHBOARD.md`
