# Schema — Config (Vehicles, Goals, Permissions, i18n)

Hub: [[OnTime]] · Features: [[GOALS-PERMISSIONS]] · [[I18N]].

> Note: `translation_entries` (and the proposed `translation_locales`) are currently **unused** — i18n is served from hardcoded dictionaries in `I18nController.cs`. See [[I18N]].

## vehicle_brands
Global "shelf" — shared, not per-tenant. No public endpoint creates rows under it directly
anymore (see [[USER-BRANDS]]); `ManagerOnly` can still create/delete the brand itself.
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| name | TEXT UNIQUE | |
| logo_url | TEXT nullable | |

## vehicle_models / vehicle_model_versions (global shelf — redesigned 2026-06-27)
Unchanged shape, but now exist **only as seed/template data** to clone from — no controller
writes to these anymore. A user's own catalog lives in `user_vehicle_models`/
`user_vehicle_versions` below; `ProposalVehicle`/`Sale` no longer FK here.
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| brand_id | UUID FK vehicle_brands | |
| name, version | TEXT | |
| year | INT nullable | |
| fuel_type | INT | 0=Petrol 1=Diesel 2=Electric 3=Hybrid 4=PluginHybrid 5=LPG 6=Other |
| base_price | NUMERIC nullable | |
| image_url | TEXT nullable | |

`vehicle_model_versions`: `model_id` FK, `name`, `external_colors`/`internal_colors` (serialized
arrays via `ColorArrayHelper`).

## user_vehicle_models / user_vehicle_versions ✅ done (2026-06-27) — see [[USER-BRANDS]]
The per-user catalog. Same column shape as `vehicle_models`/`vehicle_model_versions` above, plus:
| column | type | notes |
|--------|------|-------|
| user_id | UUID FK users | cascade delete |
| vehicle_brand_id | UUID FK vehicle_brands | cascade delete — which global brand this was cloned from/grouped under |

`ProposalVehicle.ModelId`/`VersionId` and `Sale.ModelId` FK here (Restrict on delete — blocked at
the app layer first with `VEHICLE_MODEL_IN_USE`, see [[USER-BRANDS]]).

## brand_vehicle_brands ✅ done (2026-06-27, replaces user_vehicle_brands) — see [[USER-BRANDS]]
| column | type | notes |
|--------|------|-------|
| id | UUID PK | own surrogate key (BaseEntity), not composite |
| brand_id | UUID FK brands | cascade delete — the company's Filial, not the car brand |
| vehicle_brand_id | UUID FK vehicle_brands | cascade delete |
| unique index (brand_id, vehicle_brand_id) | | many-to-many: car brands the **Filial** sells, Manager/Admin-managed. Allowing a brand lazily clones it into each user's own catalog the first time they view/create a model for it; disallowing hides (never deletes) it everywhere for that Filial's users. |

## user_brand_memberships ✅ done (2026-06-27) — see [[USER-BRANDS]], [[ARCHITECTURE]]
| column | type | notes |
|--------|------|-------|
| id | UUID PK | own surrogate key (BaseEntity), not composite |
| user_id | UUID FK users | cascade delete |
| brand_id | UUID FK brands | cascade delete |
| unique index (user_id, brand_id) | | many-to-many: which Filiais a user may switch into via `POST /api/users/me/switch-brand`. `users.company_id`/`brand_id` (and the JWT's `cid`/`bid`) stay the single "currently active" Filial — membership doesn't change that claim shape. |

## lead_source_options ✅ updated (2026-06-30, was per-company, now per-user) — see [[ROADMAP]]
| column | type | notes |
|--------|------|-------|
| id | UUID PK | own surrogate key (BaseEntity) |
| user_id | UUID FK users | cascade delete (changed from `company_id` on 2026-06-30) |
| code | int | the value stored on `clients.lead_source` — unique per user, not global |
| name | varchar(100) | free text, editable by any authenticated user for their own list |
| unique index (user_id, code) | | every new user (Manager or Salesperson) is seeded with 6 defaults (codes 0-5): Stand, Telefone, Instagram, Facebook, Recomendação, Outro at registration. Any authenticated user can manage their own list via `/api/lead-sources` (no ManagerOnly restriction). `clients.lead_source` itself is unchanged (`int`, no FK). |

## user_goals ✅ updated (2026-06-30, date range removed — period-based rolling window)
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK | |
| metric_type | INT | 0=NewClients 1=Sales 2=Proposals 3=ConversionRate |
| period | INT | 0=Daily 1=Weekly 2=Monthly 3=Annual |
| target_value | NUMERIC NOT NULL | |
| show_on_dashboard | BOOL DEFAULT FALSE | pins the goal to the Dashboard's "Os Meus Objetivos" widget |
| is_active | BOOL DEFAULT TRUE | soft delete |
> `start_date`/`end_date` removed on 2026-06-30. Progress is computed against the **current calendar period** derived from `DateTimeOffset.UtcNow` (weeks start Monday). No user-visible date fields remain.|

## menu_item_permissions
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| role | INT | 0=Salesperson 1=Manager |
| route_key | TEXT | e.g. "/clients", "/brands" — **never "/admin"** (2026-06-25: cross-tenant platform-admin access is a fixed role==2 constant, not a per-company configurable permission; a one-time startup cleanup deletes any pre-existing "/admin" row) |
| can_view, can_create, can_edit, can_delete | BOOL | all DEFAULT TRUE |
| UNIQUE(role, route_key) | | |

## error_logs *(added 2026-06-25)* — see [[SECURITY]]
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| status_code | INT | HTTP status returned (400/403/404/409/500/...) |
| error_code | TEXT | `ApiErrorCatalog` code, or "CONFLICT"/"INTERNAL_ERROR" for the two non-`ApiException` paths |
| message | TEXT(1000) | |
| details | TEXT(2000) nullable | full exception `.ToString()` for unhandled 500s; null otherwise |
| path, method, trace_id | TEXT | from the originating `HttpContext` |
| user_id | UUID nullable | no FK — must survive the user being deleted; null for unauthenticated requests |
| index (created_at), index (user_id) | | |
Written once per error by `ErrorHandlingMiddleware` (every `ApiException`, DB conflict, and
unhandled exception) — there is no other write path. `GET /api/admin/error-logs` (AdminOnly) reads it.

## translation_entries
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| locale_id | UUID FK translation_locales | new FK after i18n refactor |
| key | TEXT NOT NULL | e.g. "LABEL.CLIENT.FULL_NAME" |
| value | TEXT NOT NULL | translated string |
| is_reviewed | BOOL DEFAULT FALSE | auto-translated = false, manual = true |
| UNIQUE(locale_id, key) | | |

## translation_locales *(to add — i18n refactor)*
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| code | TEXT UNIQUE | "pt-PT", "en-GB" |
| name | TEXT | "Português (Portugal)" |
| is_base | BOOL DEFAULT FALSE | only pt-PT = true |
| is_active | BOOL DEFAULT TRUE | |

> **Note:** `translation_entries.locale` (TEXT) → would become `locale_id` FK to `translation_locales` IF the i18n rebuild moves translations to the DB. Today both are unused. See [[I18N]].

---

## Related
[[SCHEMA-AUTH]] · [[SCHEMA-PIPELINE]] · [[GOALS-PERMISSIONS]] · [[I18N]] · [[OnTime]]
