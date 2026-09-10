# online_payment_gateway_Simulation

Online Payment Gateway Simulation (OPGS) — academic sandbox for end-to-end payment workflows. No real money, banks, or card networks involved.

## What's in this repo

- `OnlinePaymentGateway.docx` — Approved SRS v1.0 (Sept 10, 2026), Wiegers/IEEE-830 based, 80 functional requirements (REQ-F-001..080), NFRs, business rules BR-001..010, test catalogue TEST-001..020
- `docs/` — Markdown Requirements Report + SRS summary + PlantUML sources
- `docs/diagrams/` — UML diagrams (use case, class, sequence, activity, state machine, component, deployment)

## Analysis of existing assets

### SRS Document (OnlinePaymentGateway.docx) — COMPLETE
- Scope: merchant account + test API keys, card/UPI/net-banking simulation, authorize/capture, lifecycle states (created, pending, authorized, captured, failed, cancelled, expired, refunded), webhooks (HMAC-SHA256, retry, replay), refunds/cancellation, fraud rules, rate limits, dashboard
- Architecture: 5 layers — API → Payment Domain → Simulation Engine → Persistence (PostgreSQL + Redis) → Event/Webhook
- Design patterns identified: Strategy, Factory, Observer, Repository, State, Adapter
- Stack: Node.js 20+ / Python 3.11+, PostgreSQL 15, Redis 7, React 18, Docker, HTTPS/TLS, JSON REST
- TBDs tracked: TBD-01..08 (language choice, Redis required?, retry policy, fraud set, rate limit defaults)

### UML Diagrams (7 images) — PRESENT, needs cleanup
1. **Use Case** (`*aret 2.04.36*`): Actors — Customer(Payer), Merchant(Business), Administrator, Simulated Bank/Card Network, Notification Service. Use cases — Make Payment, Request Refund, Create Merchant Account, Configure Simulation Rules, View Payment Status, View Transactions, Generate API Keys, Cancel Payment, Configure Webhooks, Manage API Keys, View System Logs, Manage Merchants. Issues found: crossing association lines (Customer→Create Merchant Account miswired), `<<include>>` directions inverted, admin arrows floating outside box.
2. **Class** (`*erat 2.04.35*`): Merchant, Transaction, PaymentMethod (abstract) → CardPayment/UPI/NetBanking, Refund, Webhook, ApiKey, SimulationRule. Good polymorphism, minor label typos (`vaildate`, `has`/`uses` ok).
3. **Sequence — Successful card payment** (`*ereat 2.04.35*`): Customer→Merchant App→Gateway API→Simulation Engine→Simulated Bank→Webhook Service. Labels overlapping / lifelines messy — needs redraw.
4. **Activity — Transaction processing** (`*er2.04.34*`): Receive→Validate→Select Method→Run Simulation Rule→Outcome?→Success→Capture / Failure→Mark Failed / Fraud→Block→Send Webhook+Response. Clean.
5. **State Machine — Transaction lifecycle** (`*0er9-10*`): CREATED→PENDING→AUTHORIZED→CAPTURED, branches to FAILED/CANCELLED/EXPIRED/REFUND INITIATED→REFUNDED. `decline` label on CAPTURED→FAILED is wrong, should be `fail`.
6. **Component** (`*at 2.04.34*`): Client→Web Dashboard / Payment Gateway API→Simulation Engine→PostgreSQL, Redis/Cache, Notification Service, Audit & Logging. Clean.
7. **Deployment** (`*at 2.04.33*`): Client Device→Web Server→Application Server→PostgreSQL/Redis/Simulated External Services. Clean + deployment note.

Corrected PlantUML sources are in `docs/` — start with `use-case-diagram.puml`.

## Quickstart (for implementation)
1. Create merchant account → get `test_pk_*` / `test_sk_*`
2. `POST /v1/payments` with amount (smallest unit), currency (INR/USD), payment_method
3. Simulate with synthetic card/UPI test inputs → deterministic outcome (success/decline/insufficient_funds/expired_card/timeout/fraud block)
4. Observe webhook `payment.captured` / `payment.failed` (signed HMAC-SHA256) + dashboard TEST MODE indicator

## Repo layout
```
.
├── OnlinePaymentGateway.docx
├── README.md
└── docs/
    ├── Requirements-Report.md
    ├── SRS.md
    ├── use-case-diagram.puml
    └── diagrams/
```

## Safety
TEST MODE only. Synthetic credentials. No real financial network contact. See SRS §5.2.
 