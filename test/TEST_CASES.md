# Test Cases — Online Payment Gateway Simulation

Derived from `../docs/Requirements-Report.md` (Test Catalogue TEST-001..020, REQ-F-001..080) and `../docs/SRS.md`.
All tests run against the simulation engine only — no real bank/card-network calls, no real money.

Legend: **Pre** = preconditions, **Steps** = actions, **Expected** = expected result, **Req** = traced requirement ID(s).

## 1. Payment Initiation

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-001 | Create payment with valid card details | Valid `test_sk_*` key | POST `/v1/payments` with amount (smallest unit), currency, method=card, order-ref, metadata | 201, unique payment `id`, status=created/pending, response has id/status/amount/currency/created_at/next_action | F-001..010 |
| TC-002 | Reject zero amount | Valid key | POST with amount=0 | 422, validation error, no payment created | F-004 |
| TC-003 | Reject negative amount | Valid key | POST with amount=-100 | 422, validation error | F-004 |
| TC-004 | Reject malformed amount | Valid key | POST with amount="abc" | 422, validation error | F-004 |
| TC-005 | Reject invalid/unsupported currency | Valid key | POST with currency="XXX" | 422, validation error | F-005 |
| TC-006 | Reject request with bad credentials | Invalid/missing key | POST `/v1/payments` | 401 Unauthorized | F-009 |
| TC-007 | Idempotency key prevents duplicate payment | Valid key | POST twice with same `Idempotency-Key` and same payload | Second call returns the original payment (no duplicate record created) | F-010 |
| TC-008 | Idempotency key with different payload | Valid key | POST twice with same `Idempotency-Key`, different amount | 409 Conflict (or documented error), no second payment created | F-010 |

## 2. Method Simulation

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-009 | Card payment — success outcome | Payment created, method=card | Use synthetic card mapped to "success" | Payment authorized/captured per config; never contacts a real provider | F-011..015 |
| TC-010 | Card payment — decline outcome | Payment created | Use synthetic card mapped to "decline" | Status=failed, decline_code=card_declined | F-012, F-025 |
| TC-011 | Card payment — insufficient funds | Payment created | Use synthetic card mapped to "insufficient_funds" | Status=failed, decline_code=insufficient_funds | F-012, F-025 |
| TC-012 | Card payment — expired card | Payment created | Use synthetic card mapped to "expired" | Status=failed, decline_code=expired_card | F-012, F-025 |
| TC-013 | Card payment — invalid/blocked credential | Payment created | Use synthetic card mapped to "blocked"/"invalid" | Status=failed, appropriate decline_code | F-012 |
| TC-014 | UPI payment — success outcome | Payment created, method=upi | Use synthetic UPI ref mapped to "success" | Payment authorized/captured | F-011..013 |
| TC-015 | UPI payment — timeout outcome | Payment created, method=upi | Use synthetic UPI ref mapped to "timeout" | Status=failed, decline_code=timeout, respects configured artificial delay | F-012, F-025 |
| TC-016 | Net-banking payment — failure outcome | Payment created, method=netbanking | Use synthetic bank ref mapped to "fail" | Status=failed with appropriate decline_code | F-012 |
| TC-017 | Fraud-block outcome via method simulation | Payment created | Use synthetic input mapped to "fraud" | Status=failed, decline_code=fraud_risk_blocked | F-012, F-054 |
| TC-018 | Outcome mapping is configurable without code change | Admin scenario config | Change a synthetic input's mapped outcome via config, replay | New outcome observed without redeploying/recompiling | F-014 |
| TC-019 | Configurable artificial delay does not block the API | Payment created with delay=5s configured | POST payment, measure response | Sync API responds promptly (per NF perf target); outcome resolves asynchronously after ~5s | F-015, NF-perf |

## 3. Authorization & Capture

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-020 | Authorize-only payment leaves it uncaptured | Payment created, capture=manual | Authorize successfully | Status=authorized; captured amount=0/absent | F-019, F-021 |
| TC-021 | Separate capture after authorization | Payment authorized | POST capture (full amount) | Status=captured, captured_at timestamp recorded, captured amount == authorized amount | F-020, F-023 |
| TC-022 | Reject capture on ineligible payment | Payment in failed/cancelled state | POST capture | 409/422 error, payment unchanged | F-022 |
| TC-023 | Failed authorization is never reported as captured | Payment auth fails (declined) | Inspect payment | Status=failed, never transitions to captured | F-026 |
| TC-024 | Auth/capture timestamps recorded | Payment authorized then captured | Inspect payment history | Distinct `authorized_at` and `captured_at` timestamps present | F-023 |

## 4. Lifecycle & State Transitions

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-025 | Valid transition created→pending→authorized→captured | New payment, success scenario | Drive payment through full success flow | Each transition recorded with timestamp+reason; final state=captured | F-027, F-028, F-030 |
| TC-026 | Valid transition to failed | New payment, decline scenario | Drive payment to decline | State=failed with timestamp+reason | F-027, F-030 |
| TC-027 | Cancel from created/pending | Payment in created or pending state | Cancel the payment | State=cancelled; terminal | F-027 |
| TC-028 | Expiry from pending | Pending payment, exceeds configured timeout | Wait/trigger expiry | State=expired; terminal | F-027 |
| TC-029 | Reject invalid transition per transition matrix | Payment in a terminal state (e.g., captured) | Attempt an out-of-matrix transition (e.g., re-authorize) | Request rejected, state unchanged | F-029 |
| TC-030 | Terminal states are immutable | Payment in failed/cancelled/expired/refunded state | Attempt any mutating operation | Rejected; state and history unchanged | F-032 |
| TC-031 | Full transition history retrievable via API | Payment with multiple transitions | GET payment history | All transitions returned in order with timestamp+reason | F-033 |
| TC-032 | Concurrent transition attempts are safe | Payment in authorized state | Fire two concurrent capture requests | Exactly one succeeds; no double-capture, no corrupted state | F-034 |

## 5. Webhooks

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-033 | Webhook delivered on payment.authorized | Webhook endpoint configured, payment authorized | Observe webhook delivery | Event delivered with unique event_id, type=payment.authorized, valid HMAC-SHA256 signature, payload has event_id/type/created_at/api_version/data | F-035, F-037, F-038 |
| TC-034 | Webhook delivered on payment.captured | Payment captured | Observe webhook delivery | Event type=payment.captured delivered and signed correctly | F-035, F-038 |
| TC-035 | Webhook delivered on payment.failed | Payment fails | Observe webhook delivery | Event type=payment.failed delivered and signed correctly | F-035, F-038 |
| TC-036 | Webhook delivered on refund.initiated / refund.processed | Refund initiated then processed | Observe webhook deliveries | Both events delivered in order, correctly typed and signed | F-035, F-038 |
| TC-037 | Webhook delivered on payment.cancelled | Payment cancelled | Observe webhook delivery | Event type=payment.cancelled delivered | F-035 |
| TC-038 | Webhook delivery failure retried with exponential backoff | Endpoint returns non-2xx | Observe retry attempts and timing | Retries follow exponential backoff up to configured max attempts; attempts/time recorded | F-039, F-040 |
| TC-039 | Webhook exhausts retries without corrupting payment | Endpoint always fails | Wait for max retries | Delivery marked failed; payment state/history unaffected | F-042 |
| TC-040 | Manual webhook replay from dashboard | Previously delivered (or failed) event | Trigger replay from dashboard | Event re-sent, new delivery attempt recorded (status/body/attempts/time) | F-041 |
| TC-041 | Webhook signature validation | Any delivered webhook | Recompute HMAC-SHA256 over payload with shared secret | Signature matches header value | F-038 |

## 6. Refunds & Cancellation

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-042 | Full refund of a captured payment | Payment captured | POST refund, amount=full captured amount | 201, unique refund id, status recorded, payment marked refunded | F-043, F-045, F-046 |
| TC-043 | Partial refund within captured amount | Payment captured | POST refund, amount < captured amount | Refund succeeds; unrefunded balance reduced accordingly | F-044 |
| TC-044 | Reject partial refund exceeding unrefunded balance | Payment partially refunded already | POST refund exceeding remaining unrefunded amount | 422 error, no refund created | F-044, BR-006 |
| TC-045 | Reject refund on a payment that isn't captured | Payment in failed or cancelled state | POST refund | 409/422 error | F-051 |
| TC-046 | Reject refund on fully-refunded payment | Payment already fully refunded | POST refund | 409/422 error | F-051 |
| TC-047 | Idempotent refund request | Refund request with idempotency key | POST refund twice with same key | Second call returns original refund, no duplicate | F-049 |
| TC-048 | Configurable refund success/fail scenario | Scenario config set to "fail" | POST refund | Refund recorded with status=failed per configured scenario | F-048 |
| TC-049 | Cancel eligible payment (created/pending) | Payment in created/pending state | POST cancel | Payment state=cancelled | F-043 |

## 7. Fraud Rules

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-050 | Amount-based fraud rule blocks large payment | Fraud rule: amount > threshold | Create payment above threshold | Payment blocked, decline_code=fraud_risk_blocked, rule+timestamp logged | F-052, F-055, F-056 |
| TC-051 | Velocity rule blocks rapid repeated payments | Fraud rule: N payments per window | Submit >N payments from same source within window | Payment beyond limit blocked as fraud_risk_blocked | F-053, F-056 |
| TC-052 | Country/BIN mismatch flags payment | Fraud rule: mismatch detection enabled | Submit payment with mismatched country/BIN | Payment flagged/blocked per rule ("should" — soft rule) with audit entry | F-054, F-056 |
| TC-053 | Fraud rule toggled without restart | Fraud rule active | Disable rule via admin config at runtime | New payments matching the rule are no longer blocked, no service restart needed | F-057 |

## 8. API Keys

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-054 | Generate test key pair | Authenticated developer/admin | Request new key pair | Keys prefixed `test_pk_*` and `test_sk_*`; secret shown in plaintext only at creation | F-058, F-059, F-060 |
| TC-055 | Secret key not retrievable after creation | Key already generated | Attempt to fetch secret key again | Only a masked/hashed reference returned, never plaintext | F-060 |
| TC-056 | Rotate API key | Existing key | Rotate key | New key issued; old key invalidated per policy | F-061 |
| TC-057 | Revoked key rejected immediately | Active key | Revoke key, then use it | 401 Unauthorized on next request | F-062, TEST-018 |
| TC-058 | Per-merchant key isolation | Two merchant accounts, each with own keys | Use merchant A's key to access merchant B's resource | 403 Forbidden / not found — no cross-account access | F-063, TEST-019 |

## 9. Rate Limiting & Error Handling

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-059 | Exceeding rate limit returns 429 | Rate limit configured per merchant/key | Exceed configured request rate | 429 Too Many Requests with Retry-After header | F-066..068, TEST-018 |
| TC-060 | Rate limit configurable by admin | Admin access | Change rate-limit config | New limit takes effect for subsequent requests | F-069 |
| TC-061 | Simulated 5xx/provider error scenario | Scenario config = simulate 5xx | Trigger payment under this scenario | 5xx returned with stable, documented error code | F-070, F-071 |
| TC-062 | Request for nonexistent resource returns 404 | Valid key | GET `/v1/payments/{unknown-id}` | 404 Not Found | TEST-020 |

## 10. Dashboard

| ID | Title | Pre | Steps | Expected | Req |
|---|---|---|---|---|---|
| TC-063 | Transaction explorer scoped to merchant | Two merchants with payments | Log in as merchant A, list payments | Only merchant A's payments shown | F-073 |
| TC-064 | Filter transactions by status/method/date | Multiple payments with varied attributes | Apply filters | Only matching payments shown | F-074 |
| TC-065 | Search by payment ID or order reference | Known payment ID/order-ref | Search using the value | Correct payment returned | F-075 |
| TC-066 | Payment detail view completeness | Payment with history, events, refunds | Open payment details | Amount/currency/method/status/timestamps/metadata + history + webhook events + refunds all displayed | F-076 |
| TC-067 | TEST MODE banner always visible | Any dashboard page | Load dashboard | "TEST MODE" banner shown on every page | F-077, NF-Safety |
| TC-068 | Customer role cannot see admin controls | Logged in as customer/test-user role | View dashboard | No admin controls (key mgmt, fraud config, rate-limit config) visible or accessible | F-078, BR-008 |

## Coverage Summary

| Area | Test Cases | Requirement Range |
|---|---|---|
| Payment Initiation | TC-001–TC-008 | F-001..010 |
| Method Simulation | TC-009–TC-019 | F-011..018 |
| Auth/Capture | TC-020–TC-024 | F-019..026 |
| Lifecycle | TC-025–TC-032 | F-027..034 |
| Webhooks | TC-033–TC-041 | F-035..042 |
| Refund/Cancel | TC-042–TC-049 | F-043..051 |
| Fraud | TC-050–TC-053 | F-052..057 |
| API Keys | TC-054–TC-058 | F-058..065 |
| Rate Limit/Errors | TC-059–TC-062 | F-066..072 |
| Dashboard | TC-063–TC-068 | F-073..080 |

Cross-reference: this expands the Test Catalogue (TEST-001..020) in `../docs/Requirements-Report.md` §8 into individually traceable, executable test cases
