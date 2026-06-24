# OnTimeCRM — Conventions

Hub: [[OnTimeCRM]] · Architecture: [[ARCHITECTURE]] · Maintenance: [[HOW-TO-USE]].

## Language Rule
- **All code**: English (names, comments, commits).
- **All documentation** (this vault, CLAUDE.md, decisions, sessions): **English** — always.
- **Only exception** — UI-facing text: Portuguese pt-PT from `/api/i18n`, never hardcoded in the frontend.

---

## File-size limits (local-model friendly)

| Layer | Max | Split when… |
|-------|-----|-------------|
| Controller | ~80 | >2 action groups |
| Service | ~150 | >2 responsibilities |
| Repository | ~100 | >5 query methods |
| React page | ~150 | extract sub-components |
| React component | ~100 | reused or complex |
| Doc / .md | ~150 | split by area |

If a file is growing, split it before asking the local model for help.

---

## Backend (.NET)

| Element | Convention | Example |
|---------|-----------|---------|
| Classes/Methods | PascalCase | `ClientService`, `GetPagedAsync` |
| Async methods | +Async suffix | `GetByIdAsync` |
| DTOs | +Dto / +Request | `ClientDto`, `CreateClientRequest` |
| Interfaces | I prefix | `IClientService` |
| Private fields | `_camelCase` | `_repo` |
| DB functions | `fn_snake_case` | `fn_get_clients_paged` |

- **DTOs are C# `record`s.** List DTO = lean (table); detail DTO = full; `*FilterParams` = all-nullable.
- **Service** delegates to repo, throws `ApiException(ApiErrorCatalog.CODE)` for business errors.
- **Controller**: `[ApiController] [Authorize]`, inject service only, `User.GetUserId()`, return `Ok(result)`, `CancellationToken` on all actions.
- **Error codes**: `SCREAMING_SNAKE_CASE` (`CLIENT_NOT_FOUND`, `PROPOSAL_ALREADY_CLOSED`).
- **JSON**: camelCase, nulls omitted, enums as integers, always paginate (`PagedResult<T>`, default 20, max 50).
- **Data layer by complexity** (see [[ARCHITECTURE]], [[2026-05-30-data-layer]]): complex/aggregate/multi-table → stored function `fn_*`; simple single-entity CRUD → EF in the service. Never implement the same flow in both.

### PostgreSQL naming
`snake_case` everywhere; tables plural; functions `fn_verb_noun`.

---

## Frontend (React/TypeScript)

- Components/Pages: PascalCase named exports — `export function ClientsPage()`.
- API calls in `src/api/<resource>.ts`; shared types in `src/types/api.ts` (mirror backend DTOs).
- **Server state → TanStack Query**; global client state → Zustand; forms → react-hook-form + zod.
- **i18n**: always `t['LABEL.X']`, never literal strings. Enum display via `enumHelpers` → `t[proposalStatusKey(v)]`.
- **Reuse**: list page → `<ResourcePage<T>>`; CRUD form → `<CrudModal>`; enum tag → `<EnumTag>`/wrappers; pagination → `usePagination(20)`.
- Dates: `DD/MM/YYYY` via `dayjs`, formatted through `utils/dates.ts`.

---

## Docs Maintenance Rule

**This vault is the source of truth. Code follows the docs, not the reverse.**
When something changes, update the matching note before closing the session:
- Schema → `SCHEMA-*.md` · Business rule → [[DOMAIN]] · Code pattern → this file
- Feature done → [[STATUS]] · Architecture decision → `04-DECISIONS/` ([[_TEMPLATE]])
- Code diverged from a doc → fix the doc to match reality.

Full guidance: [[HOW-TO-USE]].

---

## Never do
- `dotnet ef migrations add` (no migrations)
- Implement the same flow in both a stored function and a C# service
- Hardcode Portuguese strings in the frontend
- Auto-set `SoldAt` to UtcNow
- Expose `commission` in friend-facing endpoints
- Skip layers (controller → repository directly)
- Page sizes > 50

## Related
[[ARCHITECTURE]] · [[DOMAIN]] · [[HOW-TO-USE]] · [[OnTimeCRM]]
