# 03 — Data Model

Five custom MySQL tables. All indexed for the hot path (the access check on every gated request) and using idempotency keys on the payment events table.

The schema is deliberately separate from `wp_posts` and `wp_postmeta`. Subscriptions are first-class entities, not post-meta on synthetic post objects. This makes queries fast and predictable.

---

## Tables

### `wp_crm_hub_sites`

Registered client sites in the network. One row per publisher.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INT PK | |
| `site_key` | VARCHAR (unique) | Short identifier used in URLs and API calls |
| `url` | VARCHAR | Site URL for redirect whitelist checks |
| `api_secret` | VARCHAR | Per-site secret for signing webhooks and verifying purge requests |
| `created_at` | DATETIME | |

Registration happens through the auto-pairing endpoint: a new client sends the hub's bootstrap secret, the hub creates the row and returns the per-site `api_secret`.

### `wp_crm_hub_plans`

Available subscription plans.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INT PK | |
| `plan_key` | VARCHAR (unique) | Short identifier |
| `name` | VARCHAR | Display name |
| `description` | TEXT | Public description shown on the paywall |
| `price_cents` | INT | Amount in cents to avoid float precision issues |
| `duration_days` | INT | Period length |
| `recurring` | TINYINT | 0 = one-off, 1 = recurring card |
| `site_keys` | VARCHAR | Comma-separated list of `site_key` values this plan unlocks |

A single plan can span multiple sites via `site_keys`. This is the mechanism that makes cross-site membership native rather than bolted on.

### `wp_crm_hub_subscriptions`

Active subscriptions. One row per subscription (with history preserved via `replaced` status).

| Column | Type | Notes |
|--------|------|-------|
| `id` | INT PK | |
| `user_id` | INT | FK to `wp_users` |
| `plan_key` | VARCHAR | FK to plans |
| `status` | VARCHAR | `active`, `cancelled`, `expired`, `replaced` |
| `started_at` | DATETIME | |
| `expires_at` | DATETIME | For recurring, this is the next renewal date |
| `viva_tx_id` | VARCHAR | Initial transaction ID, used for recurring charges |

Indexes on `(user_id, status)` and `(status, expires_at)` support the two hot queries:

1. **Access check:** "Does user X have an active subscription that covers site Y?"
2. **Cron job:** "Which subscriptions expire in the next N days? Send renewal reminders."

### `wp_crm_hub_payment_events`

Idempotent ledger of every payment event received.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INT PK | |
| `event_id` | VARCHAR (unique) | Idempotency key from Viva |
| `provider` | VARCHAR | Currently always `viva` |
| `type` | VARCHAR | `payment_created`, `payment_failed`, `reversal` |
| `amount_cents` | INT | |
| `subscription_id` | INT | FK, nullable |
| `raw_payload` | LONGTEXT | Full webhook body for auditability |
| `received_at` | DATETIME | |

The unique constraint on `event_id` is what prevents double-activation of a subscription when Viva retries a webhook. The insert either succeeds (first time) or fails with a duplicate-key error (retry), and the handler treats the latter as a no-op success.

### `wp_crm_hub_audit`

Security audit trail. Every sensitive event.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INT PK | |
| `event_type` | VARCHAR | `login`, `logout`, `purchase`, `cancellation`, `login_failed`, `pairing`, etc. |
| `user_id` | INT | Nullable (some events happen before login) |
| `ip` | VARCHAR | |
| `user_agent` | VARCHAR | |
| `details` | TEXT | JSON with event-specific context |
| `created_at` | DATETIME | |

Not just for compliance — this table is what makes it possible to answer questions like "why did this user get charged twice?" or "when did this subscription actually start?" months after the fact.

---

## Why not `wp_postmeta`?

WordPress's default answer to "store custom data" is `wp_postmeta`. It's flexible and it's what most plugins use.

For subscription state, it's the wrong choice.

- **Query cost.** A subscription check on `wp_postmeta` typically needs 5–6 meta reads (`_plan`, `_status`, `_expires`, `_started`, `_user`, possibly more). A dedicated table with proper indexes does the same work in one query.
- **Schema is invisible.** With postmeta, the "schema" is scattered across meta keys and only exists in the developer's head. With a table, `DESC wp_crm_hub_subscriptions` tells you everything.
- **Types are strings.** `wp_postmeta.meta_value` is `LONGTEXT`. Every date, integer and boolean goes through PHP's type juggling on read. A typed column doesn't.
- **Reporting is painful.** Try writing "count active subscriptions grouped by plan" against postmeta. Now try it against a table with a `plan_key` column. One is a JOIN across itself; the other is a `GROUP BY`.

The trade-off is that you can't use WordPress's built-in `WP_Query` and post-meta APIs, and you need to write your own CRUD layer. For a system this small, that's a few dozen lines of `$wpdb` calls — well worth it for the performance and clarity.
