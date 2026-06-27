# OnTime — Conventions

Hub: [[OnTime]] · Architecture: [[ARCHITECTURE]] · Maintenance: [[HOW-TO-USE]].

## Language Rule
- **All code**: English (names, comments, commits).
- **All documentation** (this vault, CLAUDE.md, decisions, sessions): **English** — always.
- **Only exception** — UI-facing text: served from `/api/i18n` in **both pt-PT and en-US**, never hardcoded in the frontend.
- **Stage names, brand names, company names, etc. are tenant data, not UI strings** — never put them in `t['...']`. Each company types its own pipeline stage names (e.g. "Venda", "Negociação") freely; they're stored in the DB and shown verbatim, same as a client's name. Only the *label* ("Etapa Atual") is translated, never the *value* the user typed.
- **Validation checklist whenever a UI string is added/changed**: add the key to both `PtPT()` and `EnUS()` in `I18nController.cs`; mirror it in `OnTime_Frontend/src/utils/i18nFallback.ts` (pt-PT only — there's no en-US fallback file, English always depends on the live API call succeeding); use `t['YOUR.KEY']` in the component, never a literal. Quick sanity check before ending a session: `curl 'http://localhost:8080/api/i18n?locale=pt-PT' | jq '.map | length'` vs `...locale=en-US...` — they should match; a mismatch means a key exists in one locale's dictionary but not the other.

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
- API calls in `src/api/<resource>.ts`; shared types are split by domain under `src/types/api-*.ts` and re-exported from `src/types/api.ts` (mirror backend DTOs).
- **Server state → TanStack Query**; global client state → Zustand; forms → react-hook-form + zod.
- **i18n**: always `t['LABEL.X']`, never literal strings. Enum display via `enumHelpers` → `t[proposalStatusKey(v)]`.
- **Reuse**: list page → `<ResourcePage<T>>`; CRUD form → `<CrudModal>`; enum tag → `<EnumTag>`/wrappers; pagination → `usePagination(20)`.
- Dates: `DD/MM/YYYY` via `dayjs`, formatted through `utils/dates.ts`.

---

## Mobile / responsive patterns

Breakpoint source of truth: `src/hooks/useBreakpoint.ts` — `useIsMobile()` returns true below 768px.
List pages render two ways from the same data: `<PagedTable>` (desktop/tablet) or `<MobileCardList>`
(mobile), both driven by `<ResourcePage>`. Detail pages use the same components at every width but
must adapt layout via `isMobile`. Rules below come from real mobile bugs found and fixed
2026-06-24 — don't reintroduce them.

| Rule | Why |
|------|-----|
| Card-style rows (`renderMobileCard` in list pages) must use AntD `<Card>`, never a `<div className="bg-white ...">`. | A hardcoded `bg-white` ignores dark mode — looked broken/jarring against the dark theme. |
| Any header row with a title + button(s) needs `flex flex-wrap items-center gap-2` on the container, and `whitespace-nowrap` on the `Typography.Title`. | Without `flex-wrap`, a non-shrinking button forces the title into its leftover width, which can make a single short word wrap letter-by-letter. The button should drop to its own line instead. |
| Any `<Descriptions>` used in a detail page must set `layout={isMobile ? 'vertical' : 'horizontal'}`. | Horizontal layout gives the value column almost no width on a narrow phone, so even a date like `24/06/2026` wraps one character per line. Vertical layout (label above value) fixes it completely. |
| Truncate long dynamic text (client/company names) in flex rows with `<strong className="truncate min-w-0">` (the `min-w-0` is required — flex children don't shrink below content size by default) and mark the sibling value/tag `whitespace-nowrap`. | Without `min-w-0`, a long name overflows into the price/status tag instead of truncating with `…`. |
| `BottomNav` (5 tabs) must keep `flex-1 min-w-0` per tab and truncate the label — never let label text force a tab wider than its even share. | Fixed-width labels pushed tabs past the viewport edge, making 2 of 5 tabs unreachable. |
| Page content horizontal padding lives once in `AppLayout`'s `<Content>` (`12px` on mobile) — don't re-add `px-4`/`p-4` page-level padding in `ResourcePage`, `MobileCardList`, or `SearchFilterBar`. | Padding was being added at 3 nested levels at once, which is why content "felt too cramped" — the fix removed the inner two and kept only `AppLayout`'s. |
| A `<Table>` shown on mobile (e.g. Dashboard's "Negócios Quentes") needs `scroll={{ x: <px> }}`. | Without it, a table wider than the viewport forces the *whole page* to shrink-to-fit, making everything (text, icons, spacing) look tiny — not just the table. |

`Sidebar.tsx` (rewritten 2026-06-24) takes `variant: 'sider' | 'drawer'` + `onNavigate?: () => void`.
Nav items are grouped (`NavGroup[]` → AntD `Menu` `type: 'group'` items), not a flat list — add new
routes to the right group (`NAV.GROUP.OVERVIEW/SALES/WORKSPACE/SETTINGS`), don't append to the end.
Any consumer that renders it inside a `Drawer` **must** pass `onNavigate` to close the drawer on
click — it doesn't close itself.

---

## Single-option auto-lock (apply app-wide, not just where already done)

**Rule:** whenever a `Select`/dropdown's option list resolves to exactly **one** choice, auto-fill
that value and disable the control — never leave a one-item dropdown sitting there waiting to be
clicked. The user explicitly wants this everywhere in the app, not case-by-case.

**Why:** if there's only one possible value, asking the user to open a dropdown and pick it is
pure friction with zero decision being made. Already applied to: brand (when the user has exactly
one assigned vehicle brand), external/internal color (when a version has exactly one of each),
version (when a model has exactly one version) — all in `VehicleProposalTable.tsx`/`NewClientPage.tsx`.

**Pattern to copy** (see `VersionSelect`/`ExternalColorSelect` in `VehicleProposalTable.tsx`):
```tsx
const locked = options.length === 1 ? options[0] : undefined
useEffect(() => {
  if (locked && value !== locked.id) onChange?.(locked.id)
}, [locked, value, onChange])
// ...
<Select disabled={!!locked} allowClear={!locked} value={value} onChange={onChange} options={...} />
```

**Apply this same pattern to any other single-option selector found later** — e.g. a client's
initial Funil stage if only one is active, a Lead Source picker if a company only configured one,
etc. Don't wait to be asked again; if you spot a dropdown that can only ever have one option in a
given context, lock it.

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
[[ARCHITECTURE]] · [[DOMAIN]] · [[HOW-TO-USE]] · [[OnTime]]
