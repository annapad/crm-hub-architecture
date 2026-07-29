# CRM Hub — Multi-site Subscription Platform

**A case study of a custom WordPress subscription platform I built as an alternative to WooCommerce Subscriptions for networks of digital publishers.**

> This repository contains the architecture write-up, data model, API reference, and security notes for a system I designed and shipped in production. **The source code lives in a private repository owned by the client agency.** Everything here is documentation — enough to explain how the system works and why the decisions were made, without exposing proprietary code.

---

## The problem

A digital agency needed a subscription system for a small network of independent WordPress sites. One reader account should unlock premium content across every site in the network. The obvious candidate — WooCommerce Subscriptions — didn't fit for a few reasons:

- **Cross-site subscriptions aren't a first-class concept.** WCS is built around a single WooCommerce store. Multi-site membership requires stitching together external tooling (SSO plugins, cross-site auth, ad-hoc post filtering) and hoping it holds up.
- **Licensing costs.** WCS is a yearly paid extension. Across three sites, that's a recurring line item for a feature that isn't even solving the core requirement.
- **Surface area.** Bringing in WooCommerce and Subscriptions to run a subscription flow imports a very large plugin ecosystem for what is, at its heart, a small custom problem.

The alternative was to build the exact thing needed and skip everything else.

---

## The system, in one paragraph

**CRM Hub** is a hub-and-spoke setup. A central WordPress installation (**the hub**) is the source of truth for users, plans, subscriptions, and payments. Every publisher site in the network runs a lightweight companion plugin (**the client**) that delegates all auth and access decisions back to the hub over authenticated REST calls. A single subscription unlocks premium content across every registered site. Payment is handled through Viva Wallet, including recurring cards. Auth is a hand-coded JWT-based SSO. Content is gated server-side so nothing premium reaches the browser before the reader is authorised.

Two plugins. Five indexed MySQL tables. Twelve or so REST endpoints. No external libraries.

---

## Highlights

| Area | Approach |
|------|----------|
| **Cross-site auth** | Hand-coded HS256-signed JWT, short-lived, HttpOnly cookie |
| **Content gating** | Server-side stripping via `the_content` filter — no client-side hide |
| **Payments** | Viva Wallet Smart Checkout + card tokenisation for recurring |
| **Webhook safety** | HMAC signatures + per-event idempotency keys |
| **Site pairing** | Bootstrap secret → hub issues a per-site API secret in one round-trip |
| **Admin surface** | Central dashboard for sites, plans, subscriptions, and a security audit log |

---

## Documentation

The write-up is split across these documents in the [`docs/`](./docs) folder:

- **[01 — Architecture](./docs/01-architecture.md)** — Hub-and-spoke topology, why not WooCommerce Subscriptions, and how the components talk to each other.
- **[02 — Subscriber Flow](./docs/02-subscriber-flow.md)** — End-to-end walkthrough of what happens from the moment a reader lands on a paywalled article to the moment the article unlocks.
- **[03 — Data Model](./docs/03-data-model.md)** — The five MySQL tables, indexes, and why the schema is deliberately separate from `wp_posts`/`wp_usermeta`.
- **[04 — API Reference](./docs/04-api-reference.md)** — Hub REST endpoints, client REST endpoints, and hub frontend routes.
- **[05 — Security](./docs/05-security.md)** — Every security decision with the reasoning: HttpOnly cookies, server-side gating, HMAC signatures, brute-force protection, session rotation, and more.
- **[06 — Tech Stack](./docs/06-tech-stack.md)** — Runtime, dependencies (or the lack of them), and a note on code organisation.

---

## Status

Live in production against Viva Wallet's sandbox at time of writing; production credentials switch pending. The system has been through end-to-end debugging of the full subscription flow (paywall → SSO → payment → return → unlock) and is stable.

---

## About this repository

Documentation only. No source code. Written after the fact as a portfolio piece and a way to explain the system to future collaborators.

If you want to talk about a similar problem, or you're a hiring manager looking at custom WordPress work: I'm at **annapad24@gmail.com**.

— Anna Pantelakaki
