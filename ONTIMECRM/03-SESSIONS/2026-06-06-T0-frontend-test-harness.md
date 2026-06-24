# Session — T0: Frontend test harness

**Date:** 2026-06-06
**Task:** T0 from [[AI-TASK-PLAN]]

## What changed
- Installed `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom` as devDependencies.
- Added `"test": "vitest run"` script to `package.json`.
- Changed `vite.config.ts` import to `vitest/config` and added `test: { environment: 'jsdom', globals: true }`.
- Added sample test `src/utils/format.test.ts` (trivial arithmetic assertion).

## Gate result
- `npm run test` — 1 test, 1 passed ✅
- `npm run build` — green (chunk-size warning pre-existing, non-blocking) ✅
- `dotnet test` — 81 pass / 7 fail (7 failures are pre-existing, unrelated to T0) ⚠️

## Pre-existing backend failures (noted, not introduced here)
ClientPipelineFlowTests, GeneralFlowTests×2, NotificationFlowTests×3, StageConfigFlowTests — all failing before this session.

## Docs updated
- [[STATUS]] — frontend test harness ticked
- [[AI-TASK-PLAN]] — T0 ticked ✅

## Next
T1 — Access scope helper (requires reading [[2026-05-30-access-scope-helper]] and [[ARCHITECTURE]]).
