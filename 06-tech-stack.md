# 06 — Tech Stack

## Runtime

| Layer | Choice | Notes |
|-------|--------|-------|
| PHP | 7.4+ | Runs on any modern shared host |
| WordPress | 6.x | Both hub and client are WordPress plugins |
| Database | MySQL via `$wpdb` | Prepared statements everywhere, no query builder |
| Front-end (client) | Vanilla JavaScript | No framework, no build step |
| Caching | WordPress transients | 5-minute TTL for access checks |

Deliberately conservative choices. This system runs on the kind of hosting a small publisher can afford, without a Node build pipeline, without Composer, without external caching infrastructure.

For higher-scale deployments, transients would be swapped for Redis via a WordPress object cache — no code changes needed, just a wp-config setting.

---

## Payments

**Viva Wallet** — hosted Smart Checkout for card entry, plus card tokenisation for recurring charges.

The initial payment goes through Smart Checkout and returns a transaction ID. That transaction ID becomes the "recurring reference" — future charges POST to `/api/transactions/{tx_id}` with Basic auth (merchant ID + API key). No card details ever touch the hub.

---

## OAuth

Optional Google and Facebook OAuth for account creation. Uses each provider's standard OAuth 2.0 flow with authorization code exchange.

---

## Emails

Standard `wp_mail()`. Compatible with any SMTP plugin (WP Mail SMTP, FluentSMTP, etc.).

Transactional templates in Greek (the target audience), with dynamic values for user name, plan, amount, and expiry date. Every send is logged to the email log table for debugging.

The system detects at runtime whether the host has PHP's `mail()` function available. If not (some managed hosts disable it), emails are silently skipped rather than crashing the user flow — the log still records the intended send so the admin can see what would have gone out.

---

## Code organisation

### `crm-hub/` — central plugin, roughly 3,500 lines of PHP

```
includes/
├── class-crm-hub-jwt.php          JWT encode/decode/verify
├── class-crm-hub-db.php           Schema and query layer
├── class-crm-hub-sso.php          Auth pages, SSO mint flow
├── class-crm-hub-account.php      User account, cancellation
├── class-crm-hub-payments.php     Internal payment webhook
├── class-crm-hub-viva.php         Viva Wallet integration
├── class-crm-hub-email.php        Templated transactional emails
├── class-crm-hub-audit.php        Audit logging
├── class-crm-hub-api.php          REST API surface
├── class-crm-hub-oauth.php        Google and Facebook OAuth
└── class-crm-hub-mail-safety.php  Runtime detection of disabled mail()
admin/
└── class-crm-hub-admin.php        Admin dashboard
```

### `crm-client/` — client plugin, roughly 800 lines of PHP

```
includes/
├── class-crm-client-settings.php     Settings and auto-pairing
├── class-crm-client-sso.php          Token capture, logout, cache purge
├── class-crm-client-gating.php       Server-side content gating
├── class-crm-client-content-api.php  Access and content endpoints
├── class-crm-client-shortcodes.php   Paywall, plans, subscribe shortcodes
└── class-crm-client-ad-hider.php     Ad hiding for active subscribers
assets/
└── js/crm-client.js                  Paywall renderer, JWT flow
```

Total: about 4,300 lines of PHP + ~500 lines of vanilla JavaScript.

For scale reference: `firebase/php-jwt` alone is roughly 400 lines; WooCommerce Subscriptions is well over 100,000. The full CRM Hub codebase, doing the whole subscription platform end-to-end, is smaller than most WordPress themes.

---

## Documentation

A separate 13-page PDF (in Greek, for the client) covers deployment, admin walkthrough, and operations. That document is not in this repository — it's tied to the client engagement — but this technical write-up is a distilled English version of the same content, aimed at engineers rather than operators.
