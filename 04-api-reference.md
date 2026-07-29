# 04 — API Reference

Every endpoint the system exposes. Split into three groups: hub REST, client REST, and hub frontend routes.

---

## Hub REST endpoints

Base: `hub.example.com/wp-json/crm-hub/v1/`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/plans?site=KEY` | Return plans that cover a given site. Used by client's proxy endpoint. |
| `GET` | `/access/check` | Verify JWT and check whether the subscription covers the requested site. Cached for 5 minutes. |
| `POST` | `/auth/verify` | Server-to-server token verification (used by clients). |
| `POST` | `/sites/register` | Auto-pairing endpoint. Requires bootstrap secret. Returns per-site API secret. |
| `POST` | `/webhook/payment` | Internal HMAC-signed payment event handler. |
| `GET`/`POST` | `/viva/webhook` | Viva Wallet's webhook handler. Signature verified with the configured verification key. |

### `/access/check` — the hot path

Called every time a reader hits a gated article (unless the client has a cached decision). This is the endpoint that has to be fast.

**Request:**
```
GET /wp-json/crm-hub/v1/access/check?token=JWT&site=SITE_KEY
```

**Response:**
```json
{
  "allowed": true,
  "user_id": 42,
  "plan": "network-yearly",
  "expires_at": "2027-05-30T00:00:00Z"
}
```

Under the hood, this is one indexed query against `wp_crm_hub_subscriptions` after the JWT signature is verified. The result is cached at the client for 5 minutes.

---

## Hub frontend routes

These are user-facing pages, exposed as query-string routes to avoid conflicting with the theme's rewrite rules.

| Method | Route | Purpose |
|--------|-------|---------|
| `GET` | `/?crm_login=1` | Custom login page (with optional Google/Facebook OAuth buttons) |
| `GET` | `/?crm_register=1` | Custom registration page |
| `GET` | `/?crm_plans=1` | Plans and pricing page |
| `GET` | `/?crm_account=1` | User account and subscription management |
| `GET` | `/?crm_sso=1&site=KEY&return=URL` | SSO mint endpoint — issues JWT, redirects |
| `GET` | `/?crm_order_received=1` | Post-payment confirmation page |

`wp-login.php` is never exposed to subscribers. All auth pages match the editorial design.

---

## Client REST endpoints

Base: `site-x.example.com/wp-json/crm-client/v1/`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/access` | Read the JWT cookie, ask the hub whether access is granted. Cached for 5 minutes. |
| `GET` | `/content?post_id=X` | Return the gated post body after JWT verification. |
| `GET` | `/plans` | Same-origin proxy to the hub's `/plans` endpoint. Cached 10 minutes. Avoids exposing CORS on the hub. |

### `/content` — how gating actually works

The premium body has already been stripped from the initial page response by the `the_content` filter. This endpoint is how the JS on the paywall page fetches the actual content once the reader is authorised.

**Request:**
```
GET /wp-json/crm-client/v1/content?post_id=1234
Cookie: crm_access_jwt=<JWT>
```

**Response (allowed):**
```json
{
  "allowed": true,
  "content": "<p>The actual article body...</p>"
}
```

**Response (denied):**
```json
{
  "allowed": false,
  "reason": "no_active_subscription"
}
```

The JS renders either the content into the placeholder div, or re-renders the paywall with the reason.

---

## Signed webhooks

Cross-domain messages (payment events, cache purges) are all HMAC-signed.

### Signing scheme

- Sender computes `sig = HMAC-SHA256(shared_secret, request_body)`
- Sends `X-CRM-Signature: sig` header
- Receiver computes the same HMAC over the body and compares with `hash_equals()` (constant-time compare — resists timing attacks)

### Which secret?

- **Hub → client purge messages** use the per-site `api_secret` stored in the sites table.
- **Viva → hub payment webhooks** use the Viva verification key, fetched via `GET /api/messages/config/token` on Viva's side with Basic auth.

### Idempotency

Every payment event carries a unique `event_id`. The hub's payment table has a unique constraint on it. A retry hits the constraint, the handler catches the duplicate-key error, and returns a 200 without re-activating anything.
