# Requirements Report — Online Payment Gateway Simulation

Date: 2026-09-10 | Version: 1.0 Approved | Course: Software Engineering & Design Patterns

## 1. Business Goals
Simulate end-to-end online payment gateway behaviour in a safe test environment for developers, testers, and students. No real money movement.

## 2. Stakeholders & User Classes
| User Class | Technical Level | Primary Interactions |
|---|---|---|
| Merchant / Developer | High | REST API, webhooks, dashboard |
| Customer / Test User | Low–Medium | Checkout UI |
| QA Engineer / Tester | Medium–High | Dashboard, API, scenario controls |
| Project Manager | Low | Dashboard, reports |
| System Administrator | High | Admin dashboard, configuration |

## 3. Scope — In
- Merchant account + test API key management (`test_pk_*`, `test_sk_*`, rotate/revoke)
- Payment creation `POST /v1/payments` (amount in smallest unit, currency, method, order ref, metadata, idempotency key)
- Simulated card / UPI / net-banking with deterministic outcomes: success, decline, insufficient_funds, expired_card, blocked/invalid credential, timeout, fraud block + configurable artificial delay 0–30s
- Authorize + capture (separate capture where enabled), timestamps, machine-readable decline codes
- Lifecycle states: created, pending, authorized, captured, failed, cancelled, expired, refunded + full history, transition-matrix enforcement, concurrency-safe
- Webhooks: payment.authorized/captured/failed, refund.initiated/processed, payment.cancelled; unique event_id, HMAC-SHA256 signing, exponential-backoff retry, replay, async dispatch
- Refunds: full + partial (Σ partials ≤ captured), unique refund ID, idempotent, configurable success/fail
- Fraud rules: amount-based, velocity (N/window), country/BIN mismatch; structured `fraud_risk_blocked`, rule+timestamp audit, enable/disable without restart
- Rate limiting: per-merchant/key configurable, HTTP 429 + Retry-After
- Dashboard: transaction explorer (filter status/method/date, search payment ID/ref), details (amount/currency/method/status/timestamps/metadata/history/events/refunds), webhook logs, scenario config, TEST MODE banner, WCAG 2.1 AA

## 4. Scope — Out
Real settlement, real bank/card-network calls, real money, production PCI certification, real disputes, live instruments.

## 5. Functional Summary (REQ-F-001..080)
- 4.1 Initiation (001–010): validation, unique payment ID, idempotency
- 4.2 Method simulation (011–018): catalogue of synthetic inputs, never hits real provider
- 4.3 Auth/Capture (019–026): failed auth never shown as captured
- 4.4 Lifecycle (027–034): terminal states immutable
- 4.5 Webhooks (035–042): failure never corrupts payment
- 4.6 Refund/Cancel (043–051)
- 4.7 Fraud (052–057)
- 4.8 API keys (058–065): prefix-distinguishable, secret hashed, plaintext only at creation
- 4.9 Rate limit/errors (066–072)
- 4.10 Dashboard (073–080): customer role cannot see admin controls

## 6. Nonfunctional Targets
- Perf: 95% sync API <500ms (excl. artificial delays), 100 concurrent, 200 req/min, webhook queued <1s, dispatch <2s, search <1s
- Safety: TEST MODE banner everywhere, no real credentials, no real network path
- Security: HTTPS/TLS 1.2+, hashed secrets, authN+authZ per op, input sanitization, expiring/revocable tokens, auditable key/config/auth events, synthetic-only inputs
- Quality: 99.5% demo availability, 100% transition-matrix compliance, ≥90% domain coverage, layered modular arch, first test payment <10min, Docker-portable, audit-traceable

## 7. Business Rules BR-001..010
Test keys simulation-only; merchant data isolation; unique payment ID; smallest-unit amounts; refund only from captured; Σ refunds ≤ captured; idempotency no-dup; customers blocked from admin funcs; scenarios ≠ real bank decisions.

## 8. Test Catalogue TEST-001..020
Card success/decline/insufficient/expired, UPI success/timeout, netbank fail, idempotency dedup, webhook success/fail/replay, full/partial/invalid refund, fraud block, rate-limit 429, revoked key 401, cross-account 403, 404 handling, artificial delay non-blocking.

## 9. External Interfaces
- UI: login, key mgmt, explorer, details, scenario config, webhook logs, refunds, rate-limit config
- SW: Merchant App (REST/HTTPS/JSON) ↔ Simulation Engine ↔ PostgreSQL ↔ Redis ↔ Merchant webhook (POST signed JSON) ↔ Dashboard; optional SMTP email
- Comms: JSON, HTTPS, Bearer auth, HMAC-SHA256 webhook header, standard codes 200/201/400/401/403/404/409/422/429/500, stable error codes + correlation IDs

## 10. Constraints & Assumptions
Multi-tenant isolation, REST + standard codes, deterministic scenarios, async webhooks, no secrets in logs, pattern-friendly modular code. Depends on DB/cache availability, reachable webhook endpoints for demos, academic-use only.

## 11. TBDs
TBD-01 backend lang, TBD-02 Redis required?, TBD-03 retry/backoff tuning, TBD-04 fraud set, TBD-05 default rate limit, TBD-06 throughput, TBD-07 extra currencies, TBD-08 final coverage.

Full detail: `SRS.md` + `OnlinePaymentGateway.docx`. Diagrams: `diagrams/` + `use-case-diagram.puml`.
