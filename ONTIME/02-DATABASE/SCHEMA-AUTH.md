# Schema — Auth & Users

Hub: [[OnTime]] · Domain: [[DOMAIN]] · Features: [[PAYMENTS]] · [[FRIENDS]].

## companies
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| name | TEXT | |
| tax_id, phone, email, website, address, logo_url | TEXT | all nullable |
| is_active | BOOL DEFAULT TRUE | |
| created_at, updated_at | TIMESTAMPTZ | |

## brands
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| company_id | UUID FK companies | |
| name | TEXT | |
| primary_color | TEXT | hex e.g. "#1C69D4" |
| description, phone, email, address, logo_url | TEXT | nullable |
| is_active | BOOL DEFAULT TRUE | |
| is_automotive | BOOL DEFAULT TRUE | ✅ 2026-06-29 — "Not an automotive account" toggle, see [[VEHICLES]]. When false, hides vehicle UI and relaxes the ≥1-vehicle-required rule for this Filial's proposals. |

## users
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| company_id | UUID FK nullable | optional at registration |
| brand_id | UUID FK nullable | optional at registration |
| full_name, email | TEXT | email UNIQUE, stored LOWER() |
| password_hash | TEXT | PBKDF2 |
| phone | TEXT nullable | |
| role | INT | 0=Salesperson 1=Manager |
| is_email_verified | BOOL DEFAULT FALSE | |
| last_login_at | TIMESTAMPTZ nullable | |
| account_status | INT | 0=PendingActivation 1=Active 2=Expired 3=Suspended 4=Inactive 5=Cancelled |
| plan | INT | 0=Trial 1=Monthly 2=Annual |
| subscription_status | INT | 0=Trial 1=Active 2=PastDue 3=Cancelled 4=Expired |
| trial_ends_at | TIMESTAMPTZ nullable | |
| subscription_started_at, subscription_expires_at, subscription_cancelled_at | TIMESTAMPTZ nullable | |
| grace_period_days | INT DEFAULT 3 | |
| stripe_customer_id, stripe_subscription_id | TEXT nullable | |
| is_active | BOOL DEFAULT TRUE | |

## user_subscription_payments
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK users | |
| provider | INT | 0=Stripe 1=Ifthenpay |
| method_type | INT | 0=Card 1=MBWay 2=Multibanco |
| plan | INT | 0=Trial 1=Monthly 2=Annual |
| amount | NUMERIC | |
| status | INT | 0=Pending 1=Paid 2=Failed 3=Refunded 4=Expired |
| external_id | TEXT | Stripe PaymentIntent ID or Ifthenpay RequestId |
| period_start, period_end, paid_at | TIMESTAMPTZ nullable | |

## user_public_profiles
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| user_id | UUID FK UNIQUE | |
| (fields TBD) | | public-facing aggregated stats |

## user_friendships
| column | type | notes |
|--------|------|-------|
| id | UUID PK | |
| sender_id, receiver_id | UUID FK users | |
| status | INT | 0=Pending 1=Accepted 2=Rejected |

---

## Related
[[SCHEMA-PIPELINE]] · [[SCHEMA-CONFIG]] · [[PAYMENTS]] · [[FRIENDS]] · [[DOMAIN]] · [[OnTime]]
