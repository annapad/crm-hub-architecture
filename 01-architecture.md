# 01 — Architecture

## Topology

CRM Hub is a hub-and-spoke system.

```
                  ┌─────────────────────────────┐
                  │          CRM HUB            │
                  │ hub.example.com             │
                  │─────────────────────────────│
                  │ • Users & auth              │
                  │ • Plans & subscriptions     │
                  │ • Viva Wallet               │
                  │ • Webhooks                  │
                  │ • Email engine              │
                  │ • Audit log                 │
                  │ • Admin dashboard           │
                  │ • OAuth (Google/Facebook)   │
                  └──────────────┬──────────────┘
                                 │
                     JWT · REST · Signed webhooks
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
 ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
 │ site-a.tld   │         │ site-b.tld   │         │ site-c.tld   │
 │ CRM CLIENT   │         │ CRM CLIENT   │         │ CRM CLIENT   │
 └──────────────┘         └──────────────┘         └──────────────┘
```

**The hub** is the only component that talks to payment processors, stores card tokens, or holds plain-text user data. The client sites hold nothing more than a per-site API secret and an opaque JWT cookie.

**The clients** are lightweight. They know which content is premium, and they know how to ask the hub whether the current visitor should see it. They do not make access decisions themselves.

---

## Why not WooCommerce Subscriptions?

WooCommerce Subscriptions is the default answer for "subscriptions on WordPress" and it's a mature product. It just wasn't the right shape for this problem.

| | WooCommerce Subscriptions | CRM Hub |
|---|---|---|
| **License cost (year one)** | €279 (plugin license) | €0 |
| **Cross-site subscriptions** | Not supported natively — requires add-ons and stitching | Native multi-site mapping |
| **Query cost per gated request** | 5–6 postmeta reads | 1 indexed query |
| **External dependencies** | WooCommerce core + Subscriptions extension | None |
| **Data model** | `wp_posts` + `wp_postmeta` (posts-as-orders) | 5 dedicated indexed tables |
| **Security surface** | Large (full WC ecosystem) | Minimal and focused |

The bigger point isn't cost — it's fit. WCS is a sophisticated product that solves a lot of problems this project didn't have (variable subscriptions, coupons, product variations, tax, shipping), and asks the developer to work around the ones it doesn't (multi-site membership).

Building the small, focused version took about the same time as bending WCS to fit, and produced something with a much smaller surface area and no ongoing license.

---

## Component responsibilities

### Hub

- Owns the user table (`wp_users`) — every subscriber account lives here
- Owns the plans table — what plans exist, what they cost, what sites they unlock
- Owns the subscriptions table — who has which plan, when it started, when it expires
- Owns the payment events ledger — every Viva webhook is stored with an idempotency key
- Owns the audit log — every login, purchase, cancellation is recorded
- Handles all payment processor communication (Viva Smart Checkout, tokenisation, webhooks)
- Mints and verifies all JWTs
- Provides the admin dashboard

### Client

- Registers itself with the hub on first activation (bootstrap secret → per-site API secret)
- Filters `the_content` to strip premium content server-side
- Provides a REST endpoint the front-end JS calls to check access and fetch content
- Captures the JWT after SSO redirect and stores it in an HttpOnly cookie
- Provides the paywall UI (with plans fetched from the hub via a same-origin proxy endpoint)

### Communication

- **JWT** — short-lived, HS256-signed, shared secret. Minted by the hub, consumed by both.
- **REST** — hub exposes `/wp-json/crm-hub/v1/*`, client exposes `/wp-json/crm-client/v1/*`. All authenticated.
- **Signed webhooks** — hub-to-client purge messages (for cache invalidation on logout/cancel) are HMAC-signed.

---

## Deliberate design choices

**No `wp_postmeta` for subscription state.** Subscription lookups happen on every gated request. Query cost matters. Five indexed columns beat six postmeta reads, and the schema is easier to reason about.

**Hub as single source of truth.** Client sites store no user data beyond an API secret. If a client site is compromised, the blast radius is one site's content; user PII and payment data stay on the hub.

**Zero external libraries.** The plugin runs on any WordPress 6.x install with PHP 7.4+. No Composer dependencies, no version conflicts with other plugins in the same site, no supply-chain surface.

**Server-side gating, not client-side hiding.** The premium body is stripped from the response before it reaches the browser. This is covered in detail in [05 — Security](./05-security.md).
