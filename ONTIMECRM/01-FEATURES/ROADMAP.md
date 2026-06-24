# 🎯 ROADMAP

Ordered plan of attack. Hub: [[OnTimeCRM]] · Detailed state: [[STATUS]].

> **Rule:** when an item is done, tick it `[x]` here and update [[STATUS]].
> Significant decision during a phase → add a note in `04-DECISIONS/` (see [[_TEMPLATE]]).
> **For the local AI:** execute via [[AI-TASK-PLAN]] using the test-first loop in [[AI-WORKFLOW]].

The backend is more complete than it first looked: Goals, Friends, Permissions and
Subscription endpoints all exist. The real gaps are the frontend bugs, payment
integration, and the i18n system.

---

## Phase 0 — Stabilize the frontend (NOW)

Backend is solid; the frontend has concrete bugs. Fix the house first. Full list in [[FRONTEND-ISSUES]].

- [ ] Snooze sends `snoozeUntil`; API expects `snoozedUntil` → snooze is broken
- [ ] `NotificationStatus` i18n labels swapped (2↔3) in [[I18N]]
- [ ] `NotificationsPage` reads `n.dueAt`; DTO field is `scheduledFor`
- [ ] Hardcoded pt-PT strings (NotificationDropdown, axios 403) violate [[CONVENTIONS]]
- [ ] `brandColor` not typed on `LoginResponseDto`
- [ ] **Trial lockout** (backend): PendingActivation users blocked even from GET.

→ Blocks everything else. Full bug/debt catalog in [[KNOWN-ISSUES]]; deploy-gating infra in [[BEFORE-DEPLOY]].

---

## Phase 1 — Close MVP gaps

- [ ] **Email verification** flow (post-registration). See [[SCHEMA-AUTH]].
- [ ] Audit the implemented Goals/Friends/Permissions pages end-to-end (APIs exist; verify wiring). See [[GOALS-PERMISSIONS]], [[FRIENDS]].
- [ ] Confirm dashboard performance target (<500ms). See [[DASHBOARD]].

---

## Phase 2 — Monetization

- [ ] **Stripe** — checkout + webhook. See [[PAYMENTS]].
- [ ] **Ifthenpay** — MBWay + Multibanco + callback. See [[PAYMENTS]].
- [ ] Subscription page wired to real `InitiateAsync` (currently throws `PAYMENT_PENDING`).

→ Depends on Phase 0. Unblocks real launch.

---

## Phase 3 — i18n system rebuild

Currently translations are hardcoded in `I18nController.cs`; `TranslationEntry` is dead. See [[I18N]].

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
