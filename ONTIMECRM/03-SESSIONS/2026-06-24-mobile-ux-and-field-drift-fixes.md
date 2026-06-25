# Mobile UX pass + field-name drift bug sweep — 2026-06-24

Session log (snapshot, not updated later). System: [[SESSIONS]] · Hub: [[OnTimeCRM]].

**Goal:** live browser QA pass across all frontend flows after the previous regression-fix
session; fix whatever didn't match the backend contract or looked broken, then a dedicated
mobile-responsiveness pass.

**Done:**
- **Proposal/Sale field-name drift** (frontend types had drifted from backend DTOs, found by
  testing live in Chrome, not just unit tests):
  - `ProposalListDto`/`ProposalDto.value` → renamed to `proposalValue` (backend field). Fixed in
    `types/api-proposal.ts`, `ProposalDetailPage.tsx`, `ProposalsPage.tsx`, `ClientDetailPage.tsx`.
  - `ProposalVehicleDto.brandName` → `modelBrandName`; added missing `id`/`freeTextModel`.
  - **Convert-to-Sale modal had no `finalValue` input at all** — silently sent `undefined`,
    deserialized by the backend's non-nullable `decimal FinalValue` as `0`. This is why every
    converted sale showed `0,00 €`. Added the field (+ a `commission` field, see below).
  - `ClientDto.activeProposal` — backend never returns it; removed the dead "Proposta Activa" card.
  - `StageHistoryDto`/`ClientSaleHistoryDto` (frontend) had invented field names
    (`stageName`/`notes`/`value`/`vehicleName`) that don't exist on the backend
    (`fromStageName`/`toStageName`/`obs`/`finalValue`/`modelName`). Histórico and Vendas tabs on
    `ClientDetailPage` were rendering blank — fixed to match the real contract.
- **Stage edit form was a no-op for Active/Final/Won/Lost**: `UpdateStageRequest` never had these
  4 fields at all — `StagesPage`'s toggles were silently dropped, and **`IsActive` defaulted to
  `false`** on every edit (deactivating any stage you renamed). Added the fields end-to-end;
  enforced single-`IsWon`/single-`IsLost` per user (setting one auto-demotes the previous holder;
  `STAGE_WON_AND_LOST` 422 if both requested on the same stage). Also added real `ClientCount` per
  stage (`StageRepository.GetByUserAsync`) — it didn't exist on the backend before, so the "N
  clientes" tag was always blank and the delete-button "has clients" guard never actually fired
  client-side (server-side `STAGE_HAS_CLIENTS` still protected the data, but the UI never warned).
  `StagesPage` color field switched from a raw hex `<Input>` to AntD `<ColorPicker>`.
- **Dashboard "Negócios Quentes"**: `hotDeals` was typed as `ClientListDto[]` but the backend
  returns a distinct lean shape (`fullName`/`stageName`/`stageColor`/`lastInteractionAt`, no
  `temperature`). Etapa column read the wrong field (blank); Temp. column/filter read a
  nonexistent field, so `tempKey(undefined)` resolved to the literal fallback key
  `'MSG.EMPTY_STATE.DEFAULT'`, rendering "Sem resultados." inside the cell. Added `HotDealDto`,
  fixed the column, removed the meaningless temperature filter (every row is Hot by definition).
- **VehicleProposalTable**: Marca dropdown showed blank when editing a proposal whose vehicle
  already had a `modelId` but no `brandId` (only ever set when the user has exactly one selected
  brand). Added a `BrandSelect` that infers brand from model, mirroring the existing
  `ModelSelect` pattern, without clearing the model selection.
- **Friend search with autocomplete** (feature request, not a bug): "Adicionar Amigo" only took
  an exact email before. Added `GET /api/friends/search?q=` (name/email partial match, excludes
  self, flags `alreadyFriend`/`requestPending`); `SendFriendRequestDto` now accepts `email` or
  `userId`. `FriendsPage` add-friend modal rebuilt with a debounced `AutoComplete`, hover
  `Popover` showing email/brand/company per result, confirmation chip before sending.
- **Commission**: was never collectible at Convert-to-Sale time (only after, via Sale → Editar).
  Added a `commission` field to the convert modal (`ConvertToSaleRequest.Commission` already
  existed server-side, just never wired to a form field).
- **Mobile responsiveness pass** (see new [[CONVENTIONS]] section "Mobile / responsive patterns"
  for the rules extracted from these bugs): fixed hardcoded `bg-white` cards breaking dark mode
  (Sales/Proposals/Clients/Brands), title-wraps-into-3-lines-and-button-overlaps headers (every
  `ResourcePage` list + Stages/Friends/Goals/Admin pages), `Descriptions` wrapping dates
  character-by-character on narrow screens (Proposal/Sale/Client/Subscription detail pages),
  `BottomNav` clipping 2 of 5 tabs off-screen, triple-stacked page padding, and a non-scrollable
  Dashboard table forcing the whole page to shrink-to-fit.

**Findings / gotchas:**
- The biggest class of bug this session was **frontend TypeScript types silently drifting from
  the actual backend DTO field names** — `tsc` can't catch this because `fetch`/axios responses
  are untyped at the wire boundary; the compiler trusts whatever interface you wrote. Several of
  these had existed since the types were first written and were never caught by unit tests
  (which construct fixtures from the wrong interface, so they "pass" against themselves). Live
  browser testing against the real running stack is what surfaced all of them — this is why
  the project's TDD loop step 3 ("full docker-compose restart", see [[AI-WORKFLOW]]) explicitly
  calls for testing in the browser, not just `dotnet test`/`vitest run`.
- The Chrome automation tool's viewport/resize behavior was unreliable within a session (observed
  innerWidth jumping between 1920 → 430 → 215px across consecutive calls on the same tab without
  explicit resize calls in between). When verifying mobile layout, always read
  `window.innerWidth` via `javascript_exec` right before screenshotting to confirm what width
  you're actually looking at — don't trust the screenshot alone.
- Flex-wrap items don't shrink below their content's intrinsic width unless `min-width: 0` is set
  explicitly (Tailwind `min-w-0`). This is the root cause behind both the BottomNav clipping bug
  and the page-title-wraps-into-3-lines bug — easy to reintroduce in new pages if forgotten.

**Didn't work / dead ends:**
- Tried `mcp__Claude_in_Chrome__resize_window` to emulate a phone viewport — inconsistent across
  calls in this environment; had to re-check `window.innerWidth` after every resize and sometimes
  retry before trusting a screenshot.

**Links:** [[STATUS]] (full bullet list of fixes) · [[FRONTEND-ISSUES]] (encoding bug re-investigated
and resolved as a real but unrelated mojibake-in-source-file issue, see item 7) · [[CONVENTIONS]]
(new mobile patterns section) · [[FRIENDS]]
