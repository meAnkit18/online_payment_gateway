# Software Requirements Specification — Online Payment Gateway Simulation (OPGS)

> Git-friendly mirror of `OPGS_SRS_IEEE.docx` v1.0 Approved (2026-09-10). Full authoritative text is in `OPGS_SRS_IEEE.docx` (same folder). Formatted per **IEEE Std 830-1998**. Post-v1.0 additions (DFD, elicitation record) are marked and live here until the next docx revision. TEST MODE only — no real money, banks, or card networks.

## Revision history
| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-09-10 | Project Team | Approved SRS (Wiegers/IEEE-830 based, REQ-F-001..080, BR-001..010, TEST-001..020) |
| 1.1-mirror | 2026-09-10 | Mirror | Reformat `SRS.md` to IEEE 830 structure; add DFD Appendix B.5 + elicitation record (docx frozen) |

## Table of contents (IEEE 830)
- 1 Introduction (1.1 Purpose, 1.2 Scope, 1.3 Definitions, 1.4 References, 1.5 Overview)
- 2 Overall description (2.1 Perspective, 2.2 Functions, 2.3 Users, 2.4 Constraints, 2.5 Assumptions, 2.6 Apportioning)
- 3 Specific requirements (3.1 External interfaces, 3.2 Functional REQ-F, 3.3 Performance, 3.4 Design constraints, 3.5 Attributes, 3.6 Other incl. BR)
- Appendix A Glossary | Appendix B Analysis models | Appendix C TBD + elicitation | Traceability

---

## 1. Introduction

### 1.1 Purpose
Define functional, nonfunctional, interface, security, and operational requirements of the Online Payment Gateway Simulation so developers, testers, and students can exercise end-to-end payment workflows without real financial infrastructure. This SRS covers the gateway layer between merchant applications and a simulated processing environment: request validation, auth/capture simulation, state management, events/webhooks, refunds, fraud/rate-limit simulation, and dashboard. Conventions: `REQ-F-xxx` functional, `REQ-NF-xxx` nonfunctional, `BR-xxx` business rule, `TEST-xxx` test scenario; `shall` = mandatory, `should` = recommended. All values are synthetic test values.

### 1.2 Scope
In scope: merchant account + test API keys (`test_pk_*`/`test_sk_*`, rotate/revoke); `POST /v1/payments` (smallest-unit amount, currency, method, order ref, metadata, idempotency key); simulated card/UPI/net-banking with deterministic outcomes (success, decline, insufficient_funds, expired_card, blocked/invalid, timeout, fraud block) + delay 0–30s; authorize + capture (separate where enabled) with decline codes; lifecycle states created/pending/authorized/captured/failed/cancelled/expired/refunded + history, transition-matrix enforcement, concurrency-safe; webhooks (payment.authorized/captured/failed, refund.initiated/processed, payment.cancelled) with HMAC-SHA256, retry, replay; full/partial refunds (Σ partials ≤ captured); fraud rules (amount, velocity N/window, country/BIN mismatch); rate limiting (429 + Retry-After); dashboard explorer + TEST MODE banner. Out of scope: real settlement, real bank/network calls, real money, production PCI certification, real disputes, live instruments.

### 1.3 Definitions, acronyms, abbreviations
OPGS = Online Payment Gateway Simulation; HMAC-SHA256 webhook signing; idempotency key (no-dup resubmission); TEST MODE (all flows simulated); decline codes: `card_declined`, `insufficient_funds`, `expired_card`, `timeout`, `fraud_risk_blocked`; DFD externals/stores/processes per Appendix B.5; full glossary in docx Appendix A.

### 1.4 References
Wiegers SRS template; course material (Software Engineering & Design Patterns); Dodo Payments docs, Razorpay Test Mode docs, Stripe Testing docs (conceptual inspiration only); IEEE Std 830-1998; ISO/IEC 25010; PCI DSS principles (inspiration; simulator not PCI-certified).

### 1.5 Overview
§2 overall description → §3 external interfaces → §3.2 system features (REQ-F) → §3.3–3.6 nonfunctional/BR → Appendices (models, TBDs, traceability). Reading order: sequential Introduction → Overall Description → Interfaces → Features → NFRs.

## 2. Overall description

### 2.1 Product perspective
Standalone system exposing gateway-like APIs. Five layers: API (REST/auth/validation/rate-limit) → Payment Domain (intents, auth/capture/cancel/refunds) → Simulation (method strategies, scenario registry, delays, risk) → Persistence (PostgreSQL + Redis: merchants, keys, payments, refunds, events, scenarios) → Events (create/sign/queue/retry/replay). Flow: Merchant App → Gateway API → Payment Domain → Simulation Engine → Transaction Store → Event/Webhook Service → Merchant Webhook Endpoint.

### 2.2 Product functions
Create/validate payment requests and checkout sessions; accept simulated card/UPI/net-banking; simulate auth/capture with deterministic outcomes; maintain lifecycle + history; generate/deliver/retry/replay webhooks; process full/partial refunds; simulate cancel/expiry; apply fraud/rate-limit rules; search/filter/monitor via dashboard; detailed errors + audit.

### 2.3 User classes and characteristics
| User class | Level | Interactions |
|---|---|---|
| Merchant / Developer | High | REST API, webhooks, dashboard |
| Customer / Test User | Low–Medium | Checkout UI |
| QA Engineer / Tester | Medium–High | Dashboard, API, scenario controls |
| Project Manager | Low | Dashboard, reports |
| System Administrator | High | Admin dashboard, configuration |

### 2.4 Constraints
Multi-tenant isolation; REST + standard codes; deterministic scenarios; async webhooks; no secrets in logs; modular pattern-friendly code (Strategy, Factory, Observer, Repository, State, Adapter); HTTPS/TLS 1.2+; synthetic-only inputs; Docker-portable; WCAG 2.1 AA dashboard.

### 2.5 Assumptions and dependencies
DB/cache available; merchant webhook endpoints reachable for demos; academic-use only; currency INR/USD + English; standard HTTP codes 200/201/400/401/403/404/409/422/429/500 with stable error codes + correlation IDs.

### 2.6 Apportioning of requirements
Deferred: extra currencies (TBD-07), final coverage target (TBD-08), retry/backoff tuning (TBD-03) — see Appendix C.

## 3. Specific requirements

### 3.1 External interface requirements
- **3.1.1 User interfaces:** login, key mgmt, transaction explorer (filter status/method/date, search ID/ref), details (amount/currency/method/status/timestamps/metadata/history/events/refunds), scenario config, webhook logs, refunds, rate-limit config, TEST MODE banner everywhere.
- **3.1.2 Hardware interfaces:** none beyond standard client device → web server → app server → PostgreSQL/Redis (see deployment diagram).
- **3.1.3 Software interfaces:** Merchant App (REST/HTTPS/JSON) ↔ Simulation Engine ↔ PostgreSQL ↔ Redis ↔ Merchant webhook (POST signed JSON) ↔ Dashboard; optional SMTP.
- **3.1.4 Communications interfaces:** JSON over HTTPS, Bearer auth, HMAC-SHA256 webhook header, codes 200/201/400/401/403/404/409/422/429/500.

### 3.2 Functional requirements (REQ-F-001..080)
- **3.2.1 Payment initiation (F-001..010):** `POST /v1/payments`; amount/currency/method/order-ref/metadata; smallest-unit integer; reject zero/negative/malformed; validate currency; unique payment ID; response id/status/amount/currency/created_at/next_action; 401 on bad creds; idempotency key no-dup.
- **3.2.2 Method simulation (F-011..018):** card/UPI/net-banking synthetic catalogue; outcomes success/decline/insufficient/expired/blocked/invalid/timeout/fraud; never hits real provider; configurable mapping without code change; configurable delay.
- **3.2.3 Auth/Capture (F-019..026):** auth success/fail; optional separate capture; captured ≠ authorized; reject ineligible capture; timestamps; codes `card_declined`/`insufficient_funds`/`expired_card`/`timeout`; failed ≠ captured.
- **3.2.4 Lifecycle (F-027..034):** states created/pending/authorized/captured/failed/cancelled/expired/refunded; transition matrix; timestamp+reason per transition; terminal immutable; captured→refund path; full history API; concurrency-safe.
- **3.2.5 Webhooks (F-035..042):** types payment.authorized/captured/failed, refund.initiated/processed, payment.cancelled; unique `event_id`; payload event_id/type/created_at/api_version/data; HMAC-SHA256; record status/body/attempts/time; exp-backoff retry + max; dashboard replay; failure ≠ rollback.
- **3.2.6 Refund/Cancel (F-043..051):** cancel eligible pending; full/partial (partial ≤ unrefunded); unique refund ID; response id/payment_id/amount/status/created_at; configurable success/fail; idempotent; reject failed/cancelled/fully-refunded.
- **3.2.7 Fraud (F-052..057):** amount rules; velocity N/window; country/BIN mismatch (should); `fraud_risk_blocked`; rule+timestamp logged; toggle without restart.
- **3.2.8 API keys (F-058..065):** test publishable+secret; prefix `test_pk_`/`test_sk_`; secret plaintext only at creation; rotation with invalidation; revocation → immediate 401; per-merchant isolation.
- **3.2.9 Rate limit/errors (F-066..072):** per-merchant/key configurable documented default; 429 + Retry-After; admin configurability; simulated 5xx/provider errors; stable error codes.
- **3.2.10 Dashboard (F-073..080):** merchant-scoped list; filter status/method/date; search payment ID/ref; details + history + events + refunds; TEST labels; no admin controls for customers.

### 3.3 Performance requirements (REQ-NF-001..007)
95% sync API <500ms (excl. artificial delays); 100 concurrent; 200 req/min; webhook queued <1s; dispatch <2s; search <1s. Full targets in Requirements-Report §6 + docx §5.1.

### 3.4 Design constraints
5-layer architecture (§2.1); patterns Strategy/Factory/Observer/Repository/State/Adapter; PostgreSQL 15 + Redis 7; Node 20+ / Python 3.11+; React 18; Docker; HTTPS/TLS.

### 3.5 Software system attributes
- **Safety (NF-101..105):** TEST MODE banner everywhere; no real credentials; no real network path.
- **Security (NF-201..210):** HTTPS/TLS 1.2+, hashed secrets, authN+authZ per op, input sanitization, expiring/revocable tokens, auditable key/config/auth events, synthetic-only inputs.
- **Quality:** 99.5% demo availability; 100% transition-matrix compliance; ≥90% domain coverage; first test payment <10min; audit-traceable.

### 3.6 Other requirements
Business Rules BR-001..010 (test-keys simulation-only; merchant isolation; unique payment ID; smallest-unit amounts; refund only from captured; Σ refunds ≤ captured; idempotency no-dup; customers blocked from admin; scenarios ≠ real bank decisions — full text docx §5.5); DB/schema/indexes; logging; error mapping; i18n (INR/USD, English).

## Appendix A — Glossary (summary; full text docx Appendix A)
Rate limit, fraud rule, idempotency, TEST MODE, decline codes — see §1.3.

## Appendix B — Analysis models

### B.1 Architecture
§2.1 five layers (API → Domain → Simulation → Data → Event).

### B.2 State machine
`created→pending→authorized→captured`; `created→pending→failed`; `created→failed`; `created→cancelled`; `pending→expired`; `captured→refund_initiated→refunded / refund_failed`.

### B.3 Use cases
| Use Case | Primary Actor | Precondition | Main Outcome |
|---|---|---|---|
| Create Payment | Merchant | Valid test credentials | Payment + unique ID |
| Complete Checkout | Customer | Valid checkout session | Attempt processed |
| Run Scenario | QA Engineer | Scenario available | Expected outcome |
| Receive Webhook | Merchant System | Webhook configured | Event received+verified |
| Refund Payment | Merchant | Eligible captured | Refund + event |
| Configure Fraud Rule | Administrator | Admin auth | Rule enabled/updated |
| Manage API Key | Developer/Admin | Authenticated | Key generated/rotated/revoked |
| Inspect Transaction | Developer/QA | Authorized | Txn + history shown |

### B.4 Sequence (successful card payment)
Merchant creates request → gateway authenticates/validates → creates record → customer checkout (synthetic success) → simulation selects success → auth/capture transitions → signed webhook + response. (Full steps docx B.4; diagram `../images/3-sequence-diagram.jpeg`.)

### B.5 DFD — Context / Level 0 / Level 1 (mirror-only; docx v1.0 frozen)
- Sources: `dfd-context.puml` (Context, Process 0), `dfd-level0.puml` (Level 0, 6 processes), `dfd-level1.puml` (Level 1 P2/P3/P4/P5). Notation: Gane-Sarson.
- Context: process `0 OPGS`; externals Merchant App/Developer, Customer (Test Payer), Administrator, Merchant Webhook Endpoint. Simulated Bank is internal (Simulation Engine). All flows TEST MODE.
- Level 0: `1.0 Keys/Accounts (F-058..065)`, `2.0 Payments (F-001..026)`, `3.0 Refunds/Cancel (F-043..051)`, `4.0 Fraud/Rate-limit (F-052..057,066..072)`, `5.0 Webhooks (F-035..042)`, `6.0 Dashboard (F-073..080)`.
- Level 1: `2.1 Validate → 2.2 Create (CREATED→PENDING) → 2.3 Authorize (scenario+delay) → 2.4 Capture/Fail/Expire`; `4.1 Amount → 4.2 Velocity → 4.3 BIN/country → 4.4 Rate-limit (429+Retry-After)`; `3.1 Eligibility → 3.2 Create refund ID → 3.3 Simulate → 3.4 Cancel/Expire`; `5.1 Create+Sign → 5.2 Queue/Dispatch → 5.3 Retry/Replay (failure ≠ rollback)`.
- Balancing: P2 in = payment_request + checkout_input, out = payment_response; P3 in = refund/cancel_request, out = refund_response; P5 out = webhook_event.

#### Data dictionary (core)
| Store / Flow | Contents |
|---|---|
| D1 Payments | payment_id (unique), merchant_id, amount (smallest-unit int >0), currency (INR/USD), method, status, timestamps, decline_code, history[] |
| D2 Merchants/API Keys | merchant_id, test_pk_*/test_sk_* (hashed, plaintext only at creation), isolation per merchant |
| D3 Refunds | refund_id (unique), payment_id, amount, status, idempotency |
| D4 Events/Webhook Logs | event_id (unique), type, created_at, api_version, data, signature, status/body/attempts/time |
| D5 Scenarios/Rules | method→outcome mapping, delay 0–30s, fraud/rate-limit/refund config (no restart/code change) |
| D6 Audit Log | key/config/auth/transition events + rule+timestamp |
| payment_request | amount, currency, method, order_ref, metadata, idempotency_key + Bearer auth |
| webhook_event | event_id, type, created_at, api_version, data + HMAC-SHA256 header |

### B.6 Design patterns
Strategy (Card/UPI/NetBanking), Factory (payment/refund/event), Observer (state→webhook), Repository (DB abstraction), State (lifecycle), Adapter (future providers).

## Appendix C — TBDs + elicitation + tests

### C.1 TBDs
TBD-01 backend lang, TBD-02 Redis required?, TBD-03 retry/backoff tuning, TBD-04 fraud set, TBD-05 default rate limit, TBD-06 throughput, TBD-07 extra currencies, TBD-08 final coverage.

### C.2 Requirement elicitation methods (mirror-only)
| # | Method | Applied to OPGS | Output |
|---|---|---|---|
| 1 | Stakeholder interviews | Merchant/Dev, Customer, QA, Admin, PM | User classes §2.3, REQ-F scope |
| 2 | Document analysis + benchmarking | Stripe/Razorpay/Dodo (conceptual), Wiegers, IEEE-830 | References §1.4, webhook/HMAC + TEST MODE |
| 3 | Brainstorming / workshops | Outcome catalogue + delay | REQ-F-011..018, TEST-001..020 |
| 4 | Use-case analysis | Appendix B.3 actors/flows | DFD externals/processes |
| 5 | Prototyping / walkthroughs | Checkout→capture→webhook→refund, dashboard mock | B.4, B.2, F-073..080 |
| 6 | Questionnaire (lightweight) | TBD confirmation | C.1 |

### C.3 Test catalogue TEST-001..020
Card success/decline/insufficient/expired, UPI success/timeout, netbank fail, idempotency dedup, webhook success/fail/replay, full/partial/invalid refund, fraud block, rate-limit 429, revoked key 401, cross-account 403, 404 handling, artificial delay non-blocking.

## Traceability
REQ-F → §3.2.x; REQ-NF → §3.3/3.5; BR → §3.6; TEST → C.3; DFD P1..P6 → §3.2.1..3.2.10 as listed in B.5. Diagrams: `../images/` + `use-case-diagram.puml` + `dfd-*.puml`.
