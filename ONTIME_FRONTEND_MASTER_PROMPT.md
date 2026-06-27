# OnTimeCRM Frontend Master Prompt

This file is intentionally short. The vault under `ONTIMECRM/` is the source of truth.

## Read first
- `ONTIMECRM/OnTimeCRM.md`
- `ONTIMECRM/00-PROJECT/CONVENTIONS.md`
- `ONTIMECRM/00-PROJECT/ARCHITECTURE.md`
- `ONTIMECRM/01-FEATURES/USER-BRANDS.md`
- `ONTIMECRM/01-FEATURES/VEHICLES.md`
- `ONTIMECRM/01-FEATURES/NOTIFICATIONS.md`
- `ONTIMECRM/02-DATABASE/API-REFERENCE.md`
- `ONTIMECRM/02-DATABASE/DASHBOARD.md`
- `ONTIMECRM/02-DATABASE/FRONTEND-ISSUES.md`

## Core rules
- English only in code, docs, names, and comments.
- UI text must come from `/api/i18n` via `t['KEY']`.
- Never hardcode Portuguese strings in components.
- Keep shared types in `src/types/api.ts` aligned with backend DTOs.
- Prefer generic building blocks (`ResourcePage`, `CrudModal`, `EnumTag`, etc.).

## Frontend patterns
- React 18 + TypeScript + Vite + Ant Design.
- Use TanStack Query for server state.
- Use Zustand for auth and i18n.
- Use dayjs for dates.
- Keep forms in `react-hook-form`/zod patterns where already used.
- Keep list pages responsive and avoid re-implementing pagination.

## Domain reminders
- New client flow must respect the user's selected vehicle brands.
- Vehicle pickers should respect active/inactive rules from the docs.
- Proposal detail must keep edit and conversion flows separate.
- Sale detail should be a real route, not a dashboard fallback.
- Dashboard percentages are already returned as percentage points.

## Build and test
```bash
cd OnTimeCRM_Frontend
npm run build
npm run test
```

## Notes
- Prefer small focused components.
- Keep files under the local-model line limits.
- If a shared component grows too much, split it.
