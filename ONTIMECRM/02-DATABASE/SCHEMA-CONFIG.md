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

## user_vehicle_brands *(planned — see [[USER-BRANDS]])*
| column | type | notes |
|--------|------|-------|
| user_id | UUID FK users | |
| vehicle_brand_id | UUID FK vehicle_brands | |
| PK (user_id, vehicle_brand_id) | | many-to-many: brands the user sells |

## user_goals
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK | |
| metric_type | INT | 0=NewClients 1=Sales 2=Proposals 3=ConversionRate |
| period | INT | 0=Daily 1=Weekly 2=Monthly |
| target_value | NUMERIC NOT NULL | |
| is_active | BOOL DEFAULT TRUE | soft delete |

## menu_item_permissions
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| role | INT | 0=Salesperson 1=Manager |
| route_key | TEXT | e.g. "/clients", "/brands", "/admin" |
| can_view, can_create, can_edit, can_delete | BOOL | all DEFAULT TRUE |
| UNIQUE(role, route_key) | | |

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
