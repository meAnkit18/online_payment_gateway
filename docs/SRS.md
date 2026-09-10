# SRS Summary (extracted from OnlinePaymentGateway.docx v1.0)

> Full authoritative text is in `../OnlinePaymentGateway.docx`. This file is a Git-friendly mirror of requirements IDs for traceability.

## REQ-F (functional)
- F-001..010 Payment Initiation: POST /v1/payments, amount/currency/method/order-ref/metadata, smallest-unit integer, reject zero/negative/malformed, validate currency, unique payment ID, response id/status/amount/currency/created_at/next_action, 401 on bad creds, idempotency key no-dup
- F-011..018 Method Simulation: card/UPI/netbank, synthetic catalogue, outcomes success/decline/insufficient/expired/blocked/invalid/timeout/fraud, never hits real provider, configurable mapping without code change, configurable delay
- F-019..026 Auth/Capture: auth success/fail, optional separate capture, captured ≠ authorized, reject ineligible capture, record timestamps, codes card_declined/insufficient_funds/expired_card/timeout, failed ≠ captured
- F-027..034 Lifecycle: states created/pending/authorized/captured/failed/cancelled/expired/refunded, transition matrix, timestamp+reason per transition, terminal immutable, captured→refund path, full history API, concurrency-safe
- F-035..042 Webhooks: types payment.authorized/captured/failed, refund.initiated/processed, payment.cancelled; unique event_id; payload event_id/type/created_at/api_version/data; HMAC-SHA256; record status/body/attempts/time; exp-backoff retry + max; dashboard replay; failure ≠ rollback
- F-043..051 Refund/Cancel: cancel eligible pending, full/partial (partial ≤ unrefunded), unique refund ID, response id/payment_id/amount/status/created_at, configurable success/fail, idempotent, reject failed/cancelled/fully-refunded
- F-052..057 Fraud: amount rules, velocity N/window, country/BIN mismatch (should), fraud_risk_blocked, rule+timestamp logged, toggle without restart
- F-058..065 Keys: test publishable+secret, prefix test_pk_/test_sk_, secret plaintext only at creation, rotation with invalidation policy, revocation → immediate 401, per-merchant isolation
- F-066..072 Rate limit: per-merchant/key configurable documented default, 429 + Retry-After, admin configurability, simulated 5xx/provider errors, stable error codes
- F-073..080 Dashboard: merchant-scoped list, filter status/method/date, search payment ID/ref, details amount/currency/method/status/timestamps/metadata + history + events + refunds, TEST labels, no admin controls for customers

## REQ-NF
- Perf NF-001..007, Safety NF-101..105, Security NF-201..210 — see Requirements-Report + docx §5
- Quality attributes + Business Rules BR-001..010 + DB/schema/indexes + logging + error mapping + i18n (INR/USD, English) + coverage

## Use-case table (Appendix B.3)
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

## State machine (B.2)
created→pending→authorized→captured; created→pending→failed; created→failed; created→cancelled; pending→expired; captured→refund_initiated→refunded / refund_failed

## Patterns (B.6)
Strategy (Card/UPI/NetBanking), Factory (payment/refund/event), Observer (state→webhook), Repository (DB abstraction), State (lifecycle), Adapter (future providers)

## DFD (Appendix B.5 — added post-v1.0, SRS.md mirror only)
> `OnlinePaymentGateway.docx` v1.0 Approved is frozen — DFD lives here + PlantUML sources until next SRS revision.

- Sources: `docs/dfd-context.puml` (Context, Process 0), `docs/dfd-level0.puml` (Level 0, 6 processes), `docs/dfd-level1.puml` (Level 1 detail P2/P3/P4/P5). Notation: Gane-Sarson.
- Context: single process `0 OPGS`; externals `Merchant App/Developer, Customer (Test Payer), Administrator, Merchant Webhook Endpoint`. Simulated Bank is internal (Simulation Engine), not external. All flows TEST MODE, no real network.
- Level 0: `1.0 Manage Merchants & Keys (F-058..065)`, `2.0 Process Payments (F-001..026)`, `3.0 Refunds/Cancel (F-043..051)`, `4.0 Fraud/Rate-limit (F-052..057,066..072)`, `5.0 Webhook Events (F-035..042)`, `6.0 Dashboard (F-073..080)`.
- Level 1: `2.1 Authenticate/Validate → 2.2 Create (CREATED→PENDING) → 2.3 Authorize (scenario+delay) → 2.4 Capture/Fail/Expire`; `4.1 Amount → 4.2 Velocity N/window → 4.3 BIN/country → 4.4 Rate-limit (429+Retry-After)`; `3.1 Eligibility (captured only, Σ partials ≤ captured) → 3.2 Create refund ID (idempotent) → 3.3 Simulate outcome → 3.4 Cancel/Expire`; `5.1 Create+Sign (HMAC-SHA256) → 5.2 Queue (<1s)/Dispatch (<2s) → 5.3 Exp-backoff retry + replay (failure ≠ rollback)`.
- Balancing: P2 in = payment_request + checkout_input, out = payment_response; P3 in = refund/cancel_request, out = refund_response; P5 out = webhook_event.

### Data dictionary (core)
| Store / Flow | Contents |
|---|---|
| D1 Payments | payment_id (unique), merchant_id, amount (smallest-unit int >0), currency (INR/USD), method, status, timestamps, decline_code, history[] |
| D2 Merchants/API Keys | merchant_id, test_pk_*/test_sk_* (secret hashed, plaintext only at creation), isolation per merchant |
| D3 Refunds | refund_id (unique), payment_id, amount, status (refund_initiated/refunded/refund_failed), idempotency |
| D4 Events/Webhook Logs | event_id (unique), type, created_at, api_version, data, signature, status/body/attempts/time |
| D5 Scenarios/Rules | method→outcome mapping, delay 0–30s, fraud rules, rate limits, refund success/fail (toggle without restart/code change) |
| D6 Audit Log | key/config/auth/transition events + rule+timestamp |
| payment_request | amount, currency, method, order_ref, metadata, idempotency_key + Bearer auth |
| webhook_event | event_id, type, created_at, api_version, data + HMAC-SHA256 header |
