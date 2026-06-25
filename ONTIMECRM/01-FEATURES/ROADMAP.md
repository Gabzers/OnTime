# 🎯 ROADMAP

Ordered plan of attack. Hub: [[OnTimeCRM]] · Detailed state: [[STATUS]].

> **Rule:** when an item is done, tick it `[x]` here and update [[STATUS]].
> Significant decision during a phase → add a note in `04-DECISIONS/` (see [[_TEMPLATE]]).
> **For the local AI:** execute via [[AI-TASK-PLAN]] using the test-first loop in [[AI-WORKFLOW]].

The backend is more complete than it first looked: Goals, Friends, Permissions and
Subscription endpoints all exist. The real gaps are the frontend bugs, payment
integration, and the i18n system.

---

## Phase 0 — Stabilize the frontend ✅ done (2026-06-06)

All T1–T15 tasks complete (see [[AI-TASK-PLAN]] for the full TDD log). 128 backend + 15 frontend
tests green. Highlights: snooze field fix, i18n label swap fix, trial lockout fix, hardcoded
strings removed, FOUC fix, missing i18n keys, proposal-detail back-nav, proposal vehicle
requirement, vehicle status dot/toggle. Full bug/debt catalog in [[KNOWN-ISSUES]]; deploy-gating
infra in [[BEFORE-DEPLOY]].

---

## Phase 1 — Close MVP gaps (NOW)

- [x] **F1 — User Brands filter** ✅ 2026-06-06. See [[USER-BRANDS]].
- [ ] **Email verification** flow (post-registration). See [[SCHEMA-AUTH]].
- [x] Audit the implemented Goals/Friends/Permissions pages end-to-end ✅ 2026-06-25 (full Chrome sweep + test-quality audit — see [[STATUS]]). Found and fixed: Goals date-window bug, Friends email/name-swap bug, the cross-tenant Admin-panel authz hole. See [[GOALS-PERMISSIONS]], [[FRIENDS]], [[SECURITY]].
- [ ] Confirm dashboard performance target (<500ms). See [[DASHBOARD]].
- [ ] Data-layer reconciliation (dead/drifted `fn_*` functions). See [[2026-05-30-data-layer]].

---

## Phase 2 — Monetization

- [ ] **Stripe** — checkout + webhook. See [[PAYMENTS]].
- [ ] **Ifthenpay** — MBWay + Multibanco + callback. See [[PAYMENTS]].
- [ ] Subscription page wired to real `InitiateAsync` (currently throws `PAYMENT_PENDING`).

→ Depends on Phase 0. Unblocks real launch.

---

## Phase 3 — i18n system rebuild

Translations are hardcoded in `I18nController.cs` (pt-PT + en-US, both now genuinely complete —
2026-06-25, see [[I18N]]); `TranslationEntry` is dead. Remaining work here is purely the
architecture decision below, not missing/broken translations.

- [ ] Decide: keep hardcoded dictionaries, or move to DB (`translation_entries` + `translation_locales`)?
- [ ] Missing-key detection
- [ ] Optional auto-translation via external API (DeepL/LibreTranslate)

→ Run `/grill-me` before implementing (API choice, runtime vs background).

---

## Phase 4 — V1.1 (post-launch)
- [ ] PWA + browser push notifications
- [ ] Monthly report export (PDF)
- [ ] UX adjustments from real feedback

## Phase 5 — V1.2+ (future)
- [ ] Multi-stand / groups (manager dashboard per stand)
- [ ] Salesperson invite by email
- [ ] Lead import (Standvirtual, OLX)
- [ ] SMS / WhatsApp (V2.0)

---

## Milestones

| Milestone | Criterion | Phases |
|-----------|-----------|--------|
| Frontend usable | All pages work, no known bugs | 0 |
| MVP complete | Email verification + audited features | 0–1 |
| Launchable | Payments working | 0–2 |
| Validated | 3–5 real salespeople using it | 0–4 |

---

## Related
[[STATUS]] · [[FRONTEND-ISSUES]] · [[PAYMENTS]] · [[I18N]] · [[OnTimeCRM]]
