# Digital Billing and Subscription Platform

Last updated: Stage 2, Topic 18.4 (Ordering), not yet decided

## Stage status
- Stage 1 (Project Foundation): COMPLETED
- Stage 2 (Architecture Discovery): IN PROGRESS
- Stage 3+ : NOT STARTED
- GenAI/LLM: V2 only

## Decided (do not reopen without a real contradiction)

### Domain and services
- Domain: Digital Billing and Subscription Platform
- 5 services: Customer, Subscription, Billing, Payment, Notification
- Plan belongs to Subscription (no Pricing Service in V1)
- Subscription states (initial): INACTIVE, PENDING_PAYMENT, ACTIVE
- Invoice = financial snapshot, owned by Billing
- Payment = separate bounded context, outcomes: SUCCESS / FAILED / UNKNOWN
- UNKNOWN is never auto-treated as FAILED (reconcile + idempotency)
- Notification outside the critical Saga; durable state; dedup UNIQUE(eventId, channel)

### Data
- Database per service; no shared DB, tables, cross-service SQL or FKs

### Saga
- Orchestration inside Subscription Service; no separate Saga service
- Saga state durable in Subscription DB
- Retry transient technical failures; compensate unrecoverable business failures
- Compensation state durable and retryable (e.g. refund after payment success + permanent activation failure)
- Final state machine: TO BE COMPLETED (Topic 23)

### Communication
- Client -> Gateway -> services: HTTP
- Saga -> Billing / Payment: Kafka commands
- Billing / Payment -> Saga: Kafka events
- Subscription / Payment / Billing -> Notification: Kafka events
- Command = "please do this"; Event = "this already happened"

### API and security
- API Gateway: routing, TLS, token validation, rate limiting, correlation; no business logic
- OAuth2/OIDC IdP; services also validate JWTs independently
- Service-to-service: client-credentials service tokens; never forward user JWT
- Kafka authn/authz mechanism: OPEN (SASL/OAUTHBEARER, SCRAM or mTLS)
- POST /api/v1/subscriptions -> 202 Accepted + Location; poll GET /api/v1/subscriptions/{id}
- Required headers: Authorization, Idempotency-Key, Content-Type
- Customer context derived from identity, not client-supplied customerId
- Path versioning /api/v1, /v2; lifecycle ACTIVE -> DEPRECATED -> SUNSET -> REMOVED
- API DTOs separate from domain, JPA entities, Saga state, Kafka models
- Internal Saga states, topics, orchestration are not exposed externally

### Idempotency (partial; formalized in Topic 20)
- API: Subscription owns it; records in Subscription DB; same key + same request = reuse result; same key + different request = conflict; DB uniqueness is authoritative
- Consumer: consuming service owns duplicate protection in its own DB; offsets are not idempotency keys
- Billing V1: commandId stored on Invoice; UNIQUE(commandId) and UNIQUE(subscriptionId, billingPeriod)
- Duplicate after effect exists = treated as success (offset can commit)
- Separate consumer-idempotency table only if a command has multiple independent effects

### Kafka (Topic 18, in progress)
- Topics: billing.commands.create-invoice, billing.events.invoice-created, payment.commands.process-payment, payment.events
- Topic ownership follows business capability; consumers do not co-own contracts
- Consumer group = logical service, not instance; separate groups per logical consumer
- payment.events: 3 partitions (V1), key = paymentId, no global ordering needed
- Hot-payment bottleneck and later partition increase are accepted, documented trade-offs
- Outbox: per service, same local transaction as state change; async publisher; batch claim; lease on PROCESSING; no locks held while waiting for Kafka; at-least-once publication
- Consumer flow: begin tx -> apply change -> commit tx -> commit offset; no 2PC; never commit offset before business commit
- Principle: Kafka, outbox and offsets do NOT give business exactly-once; idempotency + DB constraints do

## ACTIVE: Topic 18.4 Ordering (not decided)
Open questions:
1. What business invariant does payment lifecycle ordering protect?
2. Is partition ordering enough to guarantee valid Saga transitions? What can still break it?
3. Do consumers validate current state before applying events? What on invalid transition: reject, defer, retry, ignore?
4. Is an event sequence/version needed beyond paymentId partitioning?
5. Ordering guarantees needed per consumer (Saga vs Notification)?
6. Does the final ordering model create unnecessary coupling?

## Remaining Kafka subtopics
- 5 Message keys, 6 Delivery semantics, 7 Offset management, 8-9 Ownership: CONSOLIDATE ONLY
- 10 Schema evolution, 11 Event contracts, 12 Kafka security, 13 Retention, 14 Replay, 15 DLQ, 16 Failure scenarios, 17 Kafka complexity review: OPEN

## Remaining Stage 2 roadmap (fixed order)
19 Outbox (formalize) -> 20 Idempotency (formalize) -> 21 Retry -> 22 Resilience -> 23 Saga failure/compensation -> 24 Eventual consistency -> 25 Concurrency -> 26 Observability -> 27 Failure boundaries -> 28 Scaling boundaries -> 29 Database design -> 30 Deployment -> 31 CI/CD -> 32 Architecture docs

## Session rules
- Max 2 questions per turn, then wait
- No answers before I reason
- Under ~400 words per turn
- Architecture only; no code or versions until Stage 3
- CONSOLIDATE ONLY items: summarize and check for contradictions
- Stop at end of Stage 2; wait for "Stage 2 completed"