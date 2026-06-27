# 🏠 OnTime — Hub

> CRM SaaS for car-dealership salespeople (Portugal).
> Core value: never miss a follow-up. See [[NOTIFICATIONS]].

**This note is the entry point. Always start here.** Every note is linked from here; open the graph view to see the network.

---

## 🧭 Navigation

### Planning
- [[ROADMAP]] — what's left, in what order, by phase 🎯
- [[AI-TASK-PLAN]] — step-by-step tasks for the local AI (test-first) ✅
- [[AI-WORKFLOW]] — the red→green→verify loop every task follows 🔁
- [[STATUS]] — what's done / broken right now
- [[KNOWN-ISSUES]] — bug & tech-debt backlog (fix during dev) 🐞
- [[BEFORE-DEPLOY]] — deploy-gating infra/config only ⚠️
- [[LOCAL-AI-SETUP]] — run Ollama locally against this repo + vault 🤖
- [[HOW-TO-USE]] — how to maintain this vault and feed context to the local model

### Foundations
- [[ARCHITECTURE]] — stack, layers, critical patterns
- [[DOMAIN]] — entities, business rules, canonical enum reference
- [[CONVENTIONS]] — naming, code patterns, file-size limits
- [[SECURITY]] — auth, authorization, secrets, hardening backlog 🔒

### Database
- [[SCHEMA-AUTH]] — companies, brands, users, payments, friendships
- [[SCHEMA-PIPELINE]] — clients, stages, proposals, sales, notifications
- [[SCHEMA-CONFIG]] — vehicles, goals, permissions, i18n

### Features
- [[NOTIFICATIONS]] — ⭐ core feature (automatic follow-ups)
- [[GOALS-PERMISSIONS]] — personal goals + access control (implemented)
- [[FRIENDS]] — salesperson social network (implemented)
- [[VEHICLES]] — catalogue (brands/models/versions); seed incl. XPENG
- [[USER-BRANDS]] — per-user brand selection + vehicle filtering (planned)
- [[PAYMENTS]] — subscriptions; payment integration stubbed
- [[I18N]] — translations + multi-language plan

### Reference
- [[API-REFERENCE]] — every endpoint, grouped by resource
- [[DASHBOARD]] — KPIs and the queries behind them
- [[TESTING]] — integration test approach
- [[FRONTEND-ISSUES]] — concrete known bugs to fix

### History / memory
- `04-DECISIONS/` — timeless architecture rules (see [[_TEMPLATE]]). No session diary — Git
  history covers "what happened when."

---

## ⚡ Golden rules

1. **All code AND docs in English.** Only UI text is pt-PT, from `/api/i18n`. See [[CONVENTIONS]].
2. **DB access only via stored functions** (`fn_*`) — never LINQ for reads. See [[ARCHITECTURE]].
3. **No migrations** — EnsureCreated + auto-recreate. See [[ARCHITECTURE]].
4. **`SoldAt` is never UtcNow** — always from the request. See [[DOMAIN]].
5. **This vault is the source of truth** — if code diverges, update the note. See [[HOW-TO-USE]].

---

## 📦 Stack

| Layer | Tech |
|-------|------|
| Backend | .NET 8, ASP.NET Core, EF Core, PostgreSQL |
| Frontend | React 18, TypeScript, Vite, Ant Design 5, Zustand, TanStack Query |
| Deploy | Render.com (API) · Vercel (Front) · Supabase (DB) |
| Local AI | Ollama + qwen2.5-coder:14b (implementation) |
| Planning | Claude (architecture, tests, decisions) |

---

## 🔄 Workflow

```
Plan with Claude   →  record in [[ROADMAP]] / 04-DECISIONS
        ↓
Implement with local model (surgical context: @file only the relevant notes)
        ↓
Something changed  →  update the matching note (see [[HOW-TO-USE]])
```
