# 02 — Subscriber Flow

An end-to-end walkthrough of what actually happens when a reader hits a paywalled article and eventually reads it.

---

## 1. Reader lands on a premium article on a client site

The client's PHP layer strips the premium body from the response via the `the_content` filter. The browser receives only a placeholder `<div>` where the article body would be. **View-source cannot bypass this** — the premium HTML never entered the response in the first place.

## 2. JavaScript renders the paywall and fetches plans

The paywall UI is rendered client-side into the placeholder div. It shows a login/register button and a list of available plans.

Plans are loaded from the hub via a **same-origin proxy endpoint** on the client (`/wp-json/crm-client/v1/plans`). The client caches the response for 10 minutes in a transient. This avoids exposing CORS on the hub and gives the client control over which plans display on each site.

## 3. Reader clicks Sign in / Register → redirect to hub

The client redirects to `hub.example.com/?crm_sso=1&site=SITE_KEY&return=ARTICLE_URL`.

The `return` URL is verified against the hub's registered sites table before any redirect happens. This prevents open-redirect attacks — an attacker can't craft a link like `?return=https://phish.example.com` and have the hub redirect there.

## 4. Custom login or registration page

The hub never exposes `wp-login.php` to subscribers. All auth pages are custom, styled to match the editorial design, and rate-limited.

Brute-force protection: five failed attempts from one IP in ten minutes locks that IP out for fifteen minutes.

## 5. Plans page → Viva Smart Checkout

After registration or login, the reader lands on the hub's plans page. Selecting a plan creates a Viva order with the correct amount and a `recurring` flag, stores an order metadata record locally, and redirects the reader to Viva's hosted Smart Checkout.

## 6. Viva → webhook + return URL

Two things happen when Viva confirms payment:

- **Webhook** — Viva calls the hub's HMAC-signed webhook endpoint. The hub verifies the signature with `hash_equals` (constant-time compare) and activates the subscription. Because every payment event has a unique idempotency key, Viva's retries — which happen aggressively on transient errors — cannot double-activate the subscription.

- **Return URL** — The reader is redirected back to `hub.example.com/?crm_viva_return=1&result=success`. The hub auto-logs the reader in (they're already known) and shows the "Order Received" page.

## 7. Hub mints a JWT and redirects to the article

Once the subscription is active, the hub mints a short-lived (5-minute) HS256-signed JWT containing `user_id`, `plan`, and `site`, and redirects the reader to a client callback URL that includes the JWT.

The client plugin catches the JWT, stores it in an HttpOnly + SameSite=Lax cookie, cleans the URL, and reloads.

## 8. Article unlocks — content served via authenticated endpoint

On the reload, the client's server-side gating still strips the article body — because it doesn't know yet whether this specific reader should see it. The paywall UI, however, now sees the cookie present.

The paywall's JS calls `/wp-json/crm-client/v1/content?post_id=X`. The client's PHP handler:

1. Reads the JWT from the cookie
2. Calls the hub's `/wp-json/crm-hub/v1/access/check` endpoint to verify it (result cached for 5 minutes)
3. If access is granted, returns the article body as JSON
4. The JS renders the body into the placeholder div

The reader sees the article. Total round-trip after payment: 2–3 seconds.

---

## What happens on subsequent visits

For the next 5 minutes (JWT lifetime), no round-trip to the hub is needed for the content endpoint — the client's cached access decision is enough.

After 5 minutes, the next content request re-verifies with the hub. If the subscription is still active, the reader keeps reading. If it's been cancelled (and the client has received the signed cache-purge webhook from the hub), the reader hits the paywall again.

---

## What happens on cancellation

The hub's account page has a "Cancel subscription" button. On click:

1. The hub marks the subscription as `cancelled` and records the event in the audit log
2. The hub fires **signed cache-purge webhooks** to every client site that plan covered
3. Each client verifies the HMAC signature and deletes its cached access decision for that user
4. The next time that reader requests premium content, the check goes back to the hub, which returns `denied`, and the paywall reappears
