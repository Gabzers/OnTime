# OnTime

A CRM for car-dealership salespeople, built around one core promise: **never miss a follow-up.**

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![License](https://img.shields.io/badge/license-private-lightgrey)]()

---

## What is this?

OnTime is a multi-tenant SaaS CRM purpose-built for car dealerships: managing leads through a sales pipeline, generating proposals, recording sales, and — its differentiating feature — **automatically reminding salespeople to follow up** at the right moments (after a scheduled visit, a test drive, days after a sale). Built as a personal/professional full-stack project to apply Clean Architecture, multi-tenant access control, and integration-test-driven development end to end.

The system is split into three repositories:

| Repo | Stack | Role |
|---|---|---|
| **OnTime** *(this repo)* | Docker Compose | Orchestrates the full stack for local/demo use |
| [OnTime_Backend](https://github.com/Gabzers/OnTime_Backend) | ASP.NET Core 8, EF Core, PostgreSQL | REST API, business logic, persistence |
| [OnTime_Frontend](https://github.com/Gabzers/OnTime_Frontend) | React 18, TypeScript, Ant Design 5 | SPA client |

## Architecture at a glance

```
┌─────────────────────┐        ┌──────────────────────┐        ┌─────────────────┐
│  OnTime_Frontend  │  HTTP  │   OnTime_Backend   │  SQL   │   PostgreSQL    │
│  React + TypeScript  │ ─────▶ │  ASP.NET Core 8 API   │ ─────▶ │       16        │
│  served by Nginx     │        │  Clean Architecture   │        │                 │
└─────────────────────┘        └──────────────────────┘        └─────────────────┘
       :80                            :8080                          :5432
```

- **Backend** follows Clean Architecture (`Domain ← Application ← Infrastructure ← API`), with simple CRUD in EF Core and complex aggregate logic in PostgreSQL stored functions. No EF migrations — schema is created and reconciled at startup. Tenant isolation (Company → Brand → User) is enforced through a single `AccessScope` claims helper rather than ad-hoc checks per controller. Full details: [Backend README](https://github.com/Gabzers/OnTime_Backend).
- **Frontend** is a code-split SPA built on two generic components (`ResourcePage<T>`, `CrudModal`) that most CRUD screens configure rather than reimplement, with TanStack Query owning server state and Zustand reserved for session/theme/i18n. Every UI string comes from the backend's `/api/i18n` endpoint — there's no hardcoded copy. Full details: [Frontend README](https://github.com/Gabzers/OnTime_Frontend).
- **Database** is provisioned and reconciled automatically by the backend on startup (`DatabaseInitializer`) — no manual migration step.

## Key features

- 🔔 **Automated follow-up notifications** — stage-triggered reminders, daily digest, snooze, overdue tracking
- 📈 **Sales pipeline** — configurable stages, Client → Proposal → Sale conversion, full history
- 🏢 **Multi-tenant by design** — Company → Brand → User scoping enforced via JWT claims, not per-query tenant filters
- 🔐 **Role-based access control** — Salesperson / Manager / Admin, with per-menu-item permission flags
- 🎯 **Goals & KPIs** — dashboard metrics and personal goal tracking
- 🤝 **Social layer** — friend requests and public profiles between salespeople
- 🌍 **Localization-first** — versioned translation API, no hardcoded frontend strings (pt-PT primary, en-US fallback)
- 💳 **Subscription gating** *(in progress)* — trial/active/expired access tiers enforced by middleware; Stripe/Ifthenpay clients are stubbed and tested against WireMock, not yet wired to a live provider

## Getting started

### Prerequisites

- [Docker](https://www.docker.com/) and Docker Compose
- A `.env` file (copy from `.env.example` and fill in secrets)

### Run the full stack

```bash
git clone https://github.com/Gabzers/OnTime.git
cd OnTime
cp .env.example .env   # set POSTGRES_PASSWORD, JWT_KEY (32+ chars), ADMIN_PASSWORD
docker-compose up -d
```

This builds and starts three services:

| Service | Port | Description |
|---|---|---|
| `frontend` | `http://localhost:80` | Nginx serving the React build, proxying `/api/*` to the backend |
| `api` | `http://localhost:8080` | ASP.NET Core API (Swagger at `/swagger`) |
| `postgres` | `5434` (host) | PostgreSQL 16, persisted to a named volume |

On first boot, the backend creates the schema, seeds a base vehicle catalogue, and — if `AdminBootstrap__*` variables are set — creates a demo admin account.

### Docker Compose layout

```yaml
services:
  postgres:   # postgres:16-alpine, healthcheck-gated
  api:        # built from ./OnTime_Backend/Dockerfile
  frontend:   # built from ./OnTime_Frontend/Dockerfile, depends_on api
```

The `api` service waits for Postgres's healthcheck before starting; `frontend` waits for `api`. Each service is built from its own repository's `Dockerfile`, so the two sub-repos can also be built and run independently for focused development (see their individual READMEs).

### Local development (without full Docker)

For active development, it's usually faster to run only PostgreSQL in Docker and run the API and frontend natively:

```bash
docker-compose up -d postgres

cd OnTime_Backend/src/OnTime.API && dotnet run     # http://localhost:5085
cd OnTime_Frontend && npm install && npm run dev       # http://localhost:5173
```

See each sub-repo's README for details on testing, configuration, and architecture.

## Repository links

- [OnTime_Backend](https://github.com/Gabzers/OnTime_Backend) — REST API, Clean Architecture, integration tests
- [OnTime_Frontend](https://github.com/Gabzers/OnTime_Frontend) — React SPA, component architecture, i18n

---

*Personal/professional project by [Gabriel Proença](https://github.com/Gabzers) — Full Stack Developer (.NET / C# / SQL Server) and MSc Software Engineering student at ISEP. Built to apply Clean Architecture, multi-tenant access control, and integration-test-driven development end to end.*
