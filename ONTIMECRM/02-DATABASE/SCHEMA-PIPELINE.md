# Schema — Pipeline (Clients, Proposals, Sales, Notifications)

Hub: [[OnTimeCRM]] · Domain: [[DOMAIN]] · Core feature: [[NOTIFICATIONS]].

## client_stages
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK | per-user, not per-brand |
| name, color | TEXT | color = hex |
| order | INT | 0-based pipeline order |
| is_final, is_won, is_lost | BOOL | terminal stage flags |
| is_active | BOOL DEFAULT TRUE | |

## clients
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK users | |
| full_name | TEXT | |
| email, phone, tax_id | TEXT nullable | |
| lead_source | INT | 0=WalkIn 1=OLX 2=Standvirtual 3=Instagram 4=Referral 5=Other |
| current_stage_id | UUID FK client_stages | |
| temperature | INT | 0=Hot 1=Warm 2=Cold — auto-updated on stage change |
| last_interaction_at | TIMESTAMPTZ | updated on every stage change |
| is_active | BOOL DEFAULT TRUE | soft delete |

## client_stage_histories
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| client_id, user_id | UUID FK | |
| from_stage_id | UUID FK nullable | null on first entry |
| to_stage_id | UUID FK | |
| obs | TEXT nullable | |
| proposal_snapshot | TEXT | JSON of active proposal at moment of change |

## proposals
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| client_id, user_id | UUID FK | |
| status | INT | 0=Active 1=Won 2=Lost 3=Cancelled |
| business_type | INT | 0=DirectPurchase … |
| payment_type | INT | 0=Cash 1=Financing 2=Leasing |
| proposal_value, discount | NUMERIC nullable | |
| proposal_date | TIMESTAMPTZ | user-controlled business date, NOT system ts |
| loss_reason | INT nullable | |
| loss_notes | TEXT nullable | |
| won_at, lost_at | TIMESTAMPTZ nullable | |
| has_trade_in | BOOL DEFAULT FALSE | |
| trade_in_plate, trade_in_brand, trade_in_model | TEXT nullable | |
| trade_in_year, trade_in_km | INT nullable | |
| trade_in_estimated_value | NUMERIC nullable | |
| is_active | BOOL DEFAULT TRUE | |

## proposal_vehicles
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| proposal_id | UUID FK | |
| model_id | UUID FK nullable | null if free-text |
| free_text_model | TEXT nullable | when not in catalogue |
| is_preferred | BOOL DEFAULT FALSE | primary vehicle of interest |
| price, discount | NUMERIC nullable | per-vehicle pricing |

## sales
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| proposal_id, client_id, user_id | UUID FK | |
| model_id | UUID FK nullable | |
| free_text_model | TEXT nullable | |
| final_value | NUMERIC NOT NULL | |
| payment_type | INT | |
| sold_at | TIMESTAMPTZ NOT NULL | **ALWAYS from request — NEVER UtcNow** |
| plate, chassis, obs | TEXT nullable | |
| commission | NUMERIC nullable | **private — never expose to friends** |
| is_active | BOOL DEFAULT TRUE | |

## notifications
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK | |
| client_id, proposal_id, sale_id | UUID FK nullable | |
| trigger | INT | 0=Manual 1=StageChanged 2=SaleClosed 3=ProposalCreated |
| status | INT | 0=Pending 1=Done 2=Snoozed 3=Ignored |
| title | TEXT NOT NULL | |
| body | TEXT nullable | |
| scheduled_for | TIMESTAMPTZ NOT NULL | |
| done_at, snoozed_until | TIMESTAMPTZ nullable | |

## stage_notification_templates
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| stage_id | UUID FK | |
| title | TEXT NOT NULL | |
| days_after | INT NOT NULL | offset from stage change |
| is_enabled | BOOL DEFAULT TRUE | |

## notification_preferences
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK UNIQUE | |
| daily_digest_time | TIME DEFAULT '09:29' | |
| digest_frequency_days | INT DEFAULT 2 | |
| sale_follow_up_days | INT DEFAULT 30 | |
| digest_enabled | BOOL DEFAULT TRUE | |
| stage_change_notifications_enabled | BOOL DEFAULT TRUE | |
| sale_notifications_enabled | BOOL DEFAULT TRUE | |
| new_client_notification_days_after | INT | sentinel column for drift detection |

---

## Related
[[SCHEMA-AUTH]] · [[SCHEMA-CONFIG]] · [[NOTIFICATIONS]] · [[DOMAIN]] · [[OnTimeCRM]]
