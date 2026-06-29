# How to use this vault

Hub: [[OnTime]] · Plan: [[ROADMAP]] · State: [[STATUS]].

This Obsidian vault is the **source of truth** for OnTime. The code implements what
is documented here — not the reverse. Always enter through [[OnTime]] (everything is linked from it).

---

## Core rule: keep docs in sync with code

Whenever something changes, update the matching note **before closing the session**.

| What changed | Update |
|--------------|--------|
| New entity or column | the relevant `SCHEMA-*.md` |
| New business rule | [[DOMAIN]] |
| New code pattern | [[CONVENTIONS]] |
| Feature done / broken / started | [[STATUS]] + tick [[ROADMAP]] |
| Architecture decision | `04-DECISIONS/YYYY-MM-DD-topic.md` (copy [[_TEMPLATE]]) |
| New feature planned | `01-FEATURES/<name>.md` + link from [[ROADMAP]] and the hub |

If code has diverged from a doc, update the doc to reflect reality — never leave them out of sync.

**Verify before trusting.** When a note states an enum value, field name, or endpoint, it should match source. If unsure, read the source file and correct the note.

---

## Feeding context to the local model (qwen2.5-coder)

Surgical context beats massive context. Never pass the whole vault — only what the task needs.

```
# New backend service:
@file CONVENTIONS.md
@file DOMAIN.md
Create GoalsService following the conventions.

# Schema work:
@file CONVENTIONS.md
@file SCHEMA-PIPELINE.md
Add column X to the proposals table.
```

---

## Vault structure

```
OnTime.md           → 🏠 HUB — entry point, everything links from here

00-PROJECT/
  ARCHITECTURE.md      → stack, layers, critical patterns
  DOMAIN.md            → entities, rules, canonical enum reference
  CONVENTIONS.md       → naming, patterns, file-size limits
  STATUS.md            → done / broken
  SECURITY.md          → auth, authz, secrets, hardening backlog
  KNOWN-ISSUES.md      → bug & tech-debt backlog (fix during dev)
  BEFORE-DEPLOY.md     → deploy-gating infra/config only
  CI-CD-WORKFLOW.md    → branch/PR/CI rules, never push straight to main, merge is manual
  LOCAL-AI-SETUP.md    → Ollama + Continue against repo + vault
  HOW-TO-USE.md        → this file

01-FEATURES/
  ROADMAP.md           → 🎯 phased plan (start here to know what to do)
  NOTIFICATIONS.md · GOALS-PERMISSIONS.md · FRIENDS.md · PAYMENTS.md · I18N.md

02-DATABASE/
  SCHEMA-AUTH.md · SCHEMA-PIPELINE.md · SCHEMA-CONFIG.md

  (reference) API-REFERENCE.md · DASHBOARD.md · TESTING.md · FRONTEND-ISSUES.md

04-DECISIONS/          → architecture decisions (see _TEMPLATE) — timeless rules, not session logs
```

Every note has `[[links]]` and a "Related" footer. Use the Obsidian graph (left icon) to navigate.

---

> **No session logs.** This vault deliberately has no "what I did on day X" diary — Git history
> already covers that. Every change either updates a durable reference note (this table) or, if
> it's a real cross-cutting rule, gets its own `04-DECISIONS/` entry written as a timeless
> Rule/Why/Implies — never as a chronicle of what happened.

## Related
[[OnTime]] · [[ROADMAP]] · [[CONVENTIONS]]
