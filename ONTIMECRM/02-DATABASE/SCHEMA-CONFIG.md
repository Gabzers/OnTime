# Schema — Config (Vehicles, Goals, Permissions, i18n)

Hub: [[OnTimeCRM]] · Features: [[GOALS-PERMISSIONS]] · [[I18N]].

> Note: `translation_entries` (and the proposed `translation_locales`) are currently **unused** — i18n is served from hardcoded dictionaries in `I18nController.cs`. See [[I18N]].

## vehicle_brands
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| name | TEXT UNIQUE | |
| logo_url | TEXT nullable | |

## vehicle_models
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| brand_id | UUID FK vehicle_brands | |
| name, version | TEXT | |
| year | INT nullable | |
| fuel_type | INT | 0=Petrol 1=Diesel 2=Electric 3=Hybrid 4=PluginHybrid 5=LPG 6=Other |
| base_price | NUMERIC nullable | |
| image_url | TEXT nullable | |

## vehicle_model_versions
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| model_id | UUID FK | |
| name | TEXT NOT NULL | |
| external_colors | TEXT | serialized array (via `ColorArrayHelper`) |
| internal_colors | TEXT | serialized array |

> Not seeded → empty in the UI until configured. See [[VEHICLES]].

## user_vehicle_brands ✅ done (2026-06-06) — see [[USER-BRANDS]]
| column | type | notes |
|--------|------|-------|
| id | UUID PK | own surrogate key (BaseEntity), not composite |
| user_id | UUID FK users | cascade delete |
| vehicle_brand_id | UUID FK vehicle_brands | cascade delete |
| created_at / updated_at / is_active / notes | — | inherited from `BaseEntity` |
| unique index (user_id, vehicle_brand_id) | | many-to-many: brands the user sells. Empty for a user = no filter = all brands. |

## user_goals
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK | |
| metric_type | INT | 0=NewClients 1=Sales 2=Proposals 3=ConversionRate |
| period | INT | 0=Daily 1=Weekly 2=Monthly 3=Annual |
| target_value | NUMERIC NOT NULL | |
| start_date, end_date | TIMESTAMPTZ | `end_date` nullable — null means "end of `period`'s own window", computed at read time. Frontend (2026-06-25) no longer collects an explicit end date at all; only `start_date`, defaulted to the start of the chosen `period` (was "today", the root cause of a progress-undercounting bug). |
| show_on_dashboard | BOOL DEFAULT FALSE | pins the goal to the Dashboard's "Os Meus Objetivos" widget |
| is_active | BOOL DEFAULT TRUE | soft delete |

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
[[SCHEMA-AUTH]] · [[SCHEMA-PIPELINE]] · [[GOALS-PERMISSIONS]] · [[I18N]] · [[OnTimeCRM]]
