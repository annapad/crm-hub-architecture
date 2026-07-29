# 05 — Security

Every security decision in the system, with the reasoning. This is the section that matters most for a reviewer — it shows which threats were considered and how they were handled.

---

## Authentication and session

### HttpOnly + SameSite cookies

The JWT is stored in a cookie with `HttpOnly` and `SameSite=Lax` flags.

- **HttpOnly** — JavaScript on the page (including any injected third-party script) cannot read the cookie. This kills the most common XSS token-stealing vector.
- **SameSite=Lax** — the browser will not send the cookie on cross-site requests, which mitigates a large class of CSRF-style token replay.

Not storing the JWT in `localStorage` is a small decision that costs nothing and closes off an entire category of attacks.

### Session rotation on login

`wp_set_auth_cookie()` is called on every successful login, which rotates the session ID. This mitigates session-fixation attacks — an attacker who somehow set a cookie value before login cannot ride the session after login.

### Brute-force protection

Per-IP transient counter: five failed attempts within ten minutes lock the IP for fifteen minutes.

Not perfect (an attacker with a botnet has many IPs), but effective against the vast majority of automated login attempts, which come from single IPs.

---

## Content protection

### Server-side gating, not client-side hiding

The premium article body is removed from the response by the `the_content` filter on the client site. The browser receives only a placeholder.

**Why this matters:** most "premium content" plugins use one of these approaches:

- **CSS `display: none`** — the content is in the HTML, just hidden. `View source` reveals it. Free.
- **JS-based removal on load** — the content is in the HTML initially, then removed. `View source` still reveals it, and disabling JS keeps it visible.
- **Server-side gating** — the content never enters the response. This is what CRM Hub does.

The first two are theatre. They "hide" content in a way that anyone with browser dev tools can bypass in seconds. Server-side gating is the only approach that actually works.

The trade-off is that the paywall has to fetch the content in a second round-trip once the user is authorised. That's a small cost for actual security.

---

## Redirect safety

### Open-redirect prevention

The SSO flow includes a `return` URL that the hub redirects to after login. Without validation, an attacker could craft a link like:

```
https://hub.example.com/?crm_sso=1&return=https://phish.example.com
```

...and a reader clicking "Sign in" on the phish page would end up trusting the redirect (it came from the real hub domain).

**Mitigation:** every `return` URL is verified against the `wp_crm_hub_sites` table before any redirect happens. If the URL's hostname doesn't match a registered client site, the redirect is refused.

---

## Payment safety

### HMAC-signed webhooks

Every cross-domain message (payment events, cache purges) is HMAC-SHA256 signed with a shared secret. The receiver verifies the signature with `hash_equals()` — a constant-time comparison that resists timing-based side-channel attacks on the secret.

Without signatures, anyone who discovers the webhook URL could forge subscription activations. With them, forgery requires the secret.

### Idempotent payment events

Every payment event has a unique idempotency key from the payment provider. The `wp_crm_hub_payment_events` table has a unique constraint on this key.

**Why it matters:** Viva Wallet, like most payment providers, retries webhooks aggressively — a webhook can be delivered dozens of times over 24 hours until the server responds 2xx. Without idempotency, every retry would re-activate the subscription and could re-trigger notification emails.

With the unique constraint, the second insert throws a duplicate-key error, the handler catches it and returns 200 immediately. Retries become no-ops.

---

## CSRF protection

All form submissions use WordPress nonces (`wp_create_nonce` + `check_admin_referer`). All AJAX endpoints verify the nonce before doing anything.

Nonces are per-user, per-action, and short-lived (24 hours by default). They mitigate the whole class of cross-site request forgery attacks where a malicious page tricks a logged-in user's browser into submitting a form to CRM Hub without their consent.

---

## Auto-pairing safety

### The bootstrap secret pattern

New client sites can register themselves with the hub — no developer needed. But this endpoint is a self-registration endpoint on the public internet. Without protection, anyone could register a fake "client" and start receiving user data.

**The protection is a bootstrap secret** — a shared string configured on both the hub and the intended new client at deploy time. The client sends it once, the hub verifies it, and the hub issues a per-site API secret in response.

After that first exchange, the bootstrap secret is no longer used — all subsequent communication uses the per-site secret. If the bootstrap secret leaks, the damage is bounded: the attacker can register fake sites, but they can't decrypt any of the existing per-site secrets.

**Rotation strategy:** the bootstrap secret can be changed at any time. Existing clients are unaffected (they don't use it anymore). Only new sites paired after the rotation need the new value.

---

## Audit log

Every sensitive event is recorded in the `wp_crm_hub_audit` table with IP and user-agent:

- Login (success and failure)
- Logout
- Purchase
- Cancellation
- Site pairing
- Password reset

The primary use isn't compliance — it's diagnosability. When something looks wrong, the audit log makes it possible to reconstruct exactly what happened, in what order, from which IP, months after the fact.

---

## What is *not* claimed

Being honest about what this system does and doesn't do:

- **No end-to-end encryption.** Data in the database is stored in plain text (except passwords, which use WordPress's default bcrypt-equivalent hashing). Anyone with database access sees everything.
- **No PCI compliance work.** All card handling is delegated to Viva. The hub never sees a full card number.
- **No formal threat model.** The decisions above cover the threats I could think of and considered common. A production deployment for a serious publisher would benefit from an external security review.
- **Hand-coded JWT verification.** Writing your own JWT code is fine if you understand the tradeoffs. Every bug in verification (algorithm confusion, timing leaks in signature compare, expiration checks) is yours to own. `hash_equals()` is used for constant-time compare, but a real library gives you more eyes on the code.

The decision to hand-code JWT rather than pull in `firebase/php-jwt` was deliberate: it's about 200 lines of code, the "zero external libraries" property is meaningful for the deployment context, and the verification path is small enough to audit thoroughly. Different projects would make a different call.
