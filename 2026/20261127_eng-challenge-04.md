---
tags: Distributed systems, System design, Fintech
---
## **Daily IT Challenge #04 — Building a Correct Payment Processing Pipeline**

November 27, 2026

### **Scenario**

You are the principal engineer responsible for redesigning the payment processing backend of a multi-tenant SaaS platform.

The platform integrates with four external payment providers. Customers can create payments, authorize funds, capture authorized payments, void authorizations, and issue refunds.

The current implementation uses synchronous HTTP calls between services. It suffers from duplicate charges, inconsistent payment statuses, missing ledger entries, and difficult incident recovery.

The company wants an event-driven architecture. The system must remain financially correct despite duplicate messages, out-of-order events, network failures, process crashes, and provider outages.

Your task is to design the new architecture.

### **1. System requirements**

#### **Business operations**

The platform supports:

- Create payment
- Authorize payment
- Capture payment
- Void authorization
- Refund payment
- Query payment status
- Reconcile payment status against providers

A payment belongs to exactly one tenant and one currency.

A payment can have multiple capture and refund operations. Partial captures and partial refunds are supported.

#### **Scale**

| Metric                   | Requirement                    |
| ------------------------ | ------------------------------ |
| Payment attempts         | 8 million/day                  |
| Peak API command rate    | 15,000 requests/sec            |
| Peak webhook ingestion   | 25,000 events/sec              |
| Tenants                  | 20,000                         |
| Payment records          | 3 billion over system lifetime |
| Providers                | 4                              |
| Payment status freshness | p99 under 3 seconds            |
| Availability target      | 99.95%                         |

#### **Infrastructure constraints**

- PostgreSQL is the primary transactional database.
- Kafka is the event streaming platform.
- Services run in Kubernetes.
- Services are written in Python and Go.
- Each tenant shares the same logical platform, database infrastructure, and application services.
- Each provider has different API semantics and webhook formats.
- The platform operates in one primary region with multi-AZ deployment.

### **2. External provider behavior**

You cannot control provider behavior.

Assume the following:

1. A provider can send the same webhook multiple times.
2. Webhooks can arrive out of order.
3. A provider can send a `CAPTURED` event before the corresponding `AUTHORIZED` event.
4. A webhook can arrive several hours after the original operation.
5. A provider can return HTTP 200 for a request while the response body indicates failure.
6. A request can time out after the provider has successfully processed it.
7. Provider APIs support idempotency keys, but retain them for only 48 hours.
8. Provider event IDs are unique only within a provider merchant account.
9. Providers may temporarily become unavailable.
10. A provider's status query API is rate-limited.

The platform must support webhook replay and manual reconciliation for historical payments.

### **3. Financial correctness requirements**

These requirements are non-negotiable.

**Payment correctness**

- A payment must never be captured beyond its authorized amount.
- Total refunds must never exceed the captured amount.
- A voided authorization must not subsequently become capturable.
- Duplicate commands must not cause duplicate external financial operations.
- A timeout must not automatically be interpreted as a failed payment.

**Ledger correctness**

The platform maintains an immutable double-entry ledger.

- Every posted journal must balance to zero within its currency.
- A financial operation must not create duplicate journal postings.
- Posted entries cannot be updated or deleted.
- Corrections must be represented by compensating entries.
- Payment status and ledger state must not contradict each other.
- A payment can have multiple captures and refunds, each with its own financial records.

**Isolation**

A tenant must never access another tenant's payments, events, or ledger entries.

### **4. Design challenges**

#### **A. Architecture and service boundaries**

Design the complete architecture.

Identify the responsibilities of:

- Payment API
- Payment orchestration service
- Provider adapter services
- Webhook ingestion service
- Ledger service
- Reconciliation service
- Kafka
- PostgreSQL

Decide which services own which data.

Explain the consistency boundaries between services.

Consider whether the ledger should share a database with payment orchestration or have an independent transactional boundary.

#### **B. Payment state machine**

Design a formal payment state machine.

Include at least these states:

- `CREATED`
- `PENDING`
- `AUTHORIZED`
- `PARTIALLY_CAPTURED`
- `CAPTURED`
- `VOIDED`
- `PARTIALLY_REFUNDED`
- `REFUNDED`
- `FAILED`
- `UNKNOWN`

Define:

1. Which transitions are valid.
2. Which transitions are terminal.
3. Whether a state can regress.
4. How partial operations affect payment state.
5. How to handle an event that contradicts the current state.
6. How to distinguish provider state from internal payment state.

Do not assume that provider events arrive in chronological order.

#### **C. Idempotency across the entire system**

A client may retry a payment request because it did not receive a response.

The API must guarantee that retries with the same idempotency key do not create additional payment operations.

However, idempotency must also work across:

- API request processing
- Database transactions
- Kafka message production
- Consumer execution
- Provider API requests
- Webhook ingestion
- Ledger posting

Define the scope, uniqueness constraints, retention periods, and behavior of idempotency keys at every layer.

Consider this sequence:

```

1. Client submits payment request.
2. Backend creates payment record.
3. Backend sends request to provider.
4. Provider successfully captures the payment.
5. Provider returns a response.
6. Backend crashes before saving the result.
7. Client retries the original request.

```

The provider has already moved money. Your system does not know that.

How does your design recover without creating another capture?

#### **D. Event processing and ordering**

Kafka delivers messages at least once in this scenario.

Design event processing that handles:

- Duplicate messages
- Out-of-order messages
- Concurrent processing of the same payment
- Consumer crashes
- Rebalances
- Poison messages
- Delayed messages
- Schema evolution

Define the Kafka partitioning strategy.

Evaluate whether partitioning by `payment_id`, `tenant_id`, or another key is appropriate.

Explain how you preserve correctness when two different events for the same payment are processed concurrently.

#### **E. Database transaction boundaries**

Design the database schema for:

- Payments
- Payment operations
- Provider requests
- Provider events
- Ledger accounts
- Ledger journals
- Ledger entries
- Idempotency records
- Reconciliation records

Specify important unique constraints and foreign keys.

Then define the transactional boundaries for these operations:

1. Create payment.
2. Initiate capture.
3. Record provider response.
4. Ingest webhook.
5. Post ledger journal.
6. Publish internal domain event.

Identify every place where a database commit and a network operation can become inconsistent.

Consider PostgreSQL transaction isolation, row-level locking, optimistic concurrency, and unique constraints.

#### **F. Reconciliation**

Assume webhook delivery is not reliable enough to be the sole source of truth.

Build a reconciliation mechanism that compares internal payment records with provider records.

Consider:

- Missing webhooks
- Lost provider responses
- Payments stuck in `PENDING`
- Internal status differing from provider status
- Duplicate ledger postings
- Provider settlement amounts differing from expected amounts
- Historical records requiring repair

Define the reconciliation schedule, batching strategy, rate-limit handling, and escalation process.

Explain how reconciliation can safely repair inconsistencies without creating new financial side effects.

### **5. Failure-injection scenarios**

Your architecture must survive each scenario below.

For every scenario, determine:

- What data has already been committed?
- What state does each service believe the payment is in?
- What happens after retry or replay?
- Can money be moved twice?
- Can ledger entries be missing or duplicated?
- How does the system converge to a correct state?

| ##  | Failure scenario                                                                     |
| -- | ------------------------------------------------------------------------------------ |
| 1  | API process crashes immediately after committing the payment record.                 |
| 2  | Provider captures successfully, but the HTTP response times out.                     |
| 3  | Kafka publishes an event, but the producer loses the acknowledgment.                 |
| 4  | Consumer commits its database transaction, then crashes before acknowledging Kafka.  |
| 5  | Two consumers process the same capture event concurrently.                           |
| 6  | `REFUNDED` webhook arrives before `CAPTURED` webhook.                                |
| 7  | Provider sends the same event 100 times.                                             |
| 8  | Database primary fails during ledger posting.                                        |
| 9  | Provider is unavailable for 45 minutes, then recovers.                               |
| 10 | A reconciliation job discovers a provider capture absent from the internal ledger.   |
| 11 | An old webhook arrives after the payment has already been refunded.                  |
| 12 | A deployment changes an event schema while old and new consumers run simultaneously. |

### **6. Performance and operational requirements**

Your design must also address production operations.

**Performance**

- Keep payment API latency low.
- Avoid excessive database lock contention.
- Prevent one high-volume tenant from starving other tenants.
- Handle webhook bursts without losing events.
- Avoid unbounded Kafka consumer lag.
- Support large historical datasets without making reconciliation disruptive.

**Observability**

Define metrics and alerts for:

- Payment success and failure rates
- Provider timeout rates
- Unknown payment outcomes
- Duplicate webhook rates
- Event processing lag
- Payment state transition failures
- Ledger imbalance detection
- Reconciliation mismatch rates
- Payments stuck in pending states
- Idempotency conflict rates

Define the correlation identifiers needed to trace a payment from API request through provider, Kafka, database, and ledger.

**Disaster recovery**

Specify the behavior when:

- A Kafka cluster becomes unavailable.
- A PostgreSQL primary becomes unavailable.
- A Kubernetes cluster loses an availability zone.
- The primary region becomes unavailable.

Define RPO and RTO targets for payment records and financial ledger data.

### **7. Evaluation criteria**

Use these criteria to assess the quality of the architecture.

| Dimension                                   | Weight |
| ------------------------------------------- | ------ |
| Financial correctness and ledger invariants | 25%    |
| Idempotency and duplicate prevention        | 20%    |
| Event ordering and concurrency              | 15%    |
| Recovery and reconciliation                 | 15%    |
| Database and transaction design             | 10%    |
| Scalability and performance                 | 10%    |
| Observability and operations                | 5%     |

A design that achieves high throughput but can double-charge a customer is incorrect.

A design that avoids duplicate charges but can permanently lose a successful provider operation is also incorrect.

### **8. The central engineering question**

**Can you build a payment system that guarantees financial correctness when every distributed component can fail independently, messages can be duplicated or reordered, and external systems do not participate in your database transactions?**

Your design must explain how it achieves correctness. It must not rely on claims of exactly-once delivery or assume that retries are harmless.

**Optional next exploration:** Build a failure-injection test matrix covering all 12 scenarios, including expected database state and ledger invariants.

---

## **Daily IT Challenge #04 — Solution: Building a Correct Payment Processing Pipeline**

Solution

Senior / Principal Engineer

Distributed systems · Fintech · PostgreSQL · Kafka

### **1. Core architectural decision**

The central design principle is:

**Exactly-once execution across distributed systems is not achievable through Kafka or database transactions alone. Financial correctness comes from durable intent, idempotent operations, explicit state transitions, immutable accounting, and reconciliation.**

I would use:

- PostgreSQL as the authoritative store for payment operations and financial records.
- Kafka as the asynchronous event distribution mechanism.
- Transactional Outbox for reliable event publication.
- Transactional Inbox for idempotent event consumption.
- Provider idempotency keys for external financial operations.
- An explicit payment state machine.
- A double-entry ledger with immutable journals.
- Reconciliation as a permanent correctness mechanism.

#### **High-level architecture**

[Payment Gateway (Stripe) System Design with Microsoft Engineer | andreyka26 tech](https://images.openai.com/static-rsc-4/a8kTOQutNanz38a7hWt2pA4_YFWN9yc6GtkQy-DjKz2dmuH56YAj_qSVqlkQEfQEOzaLy5_oZHXUkJyBhVLReVtD66drZ3gs0YKVy6UHrztbAABq18MxYE4CTDmI7UK162G4A_VowCMmmeMivRDrJBm0V14njeXlKTReVNZ5EdY?purpose=inline)

[andreyka26.com](https://andreyka26.com/payment-gateway-stripe-system-design-with-microsoft-engineer?utm_source=chatgpt.com)

```
                    Client
                      |
                      v
                Payment API
                      |
                      v
             Payment Orchestrator
                      |
             PostgreSQL (ACID)
             /       |        \
        Payments  Outbox    Operations
                      |
                      v
                 Outbox Relay
                      |
                      v
                    Kafka
                /     |      \
               v      v       v
          Provider  Ledger  Notification
          Adapter   Service  Service
             |
             v
       Payment Provider
             |
             v
       Webhook Ingestion
             |
             v
            Kafka

       Reconciliation Service
             |
             +---- PostgreSQL
             |
             +---- Provider APIs
```

One important distinction: the diagram represents logical responsibilities, not necessarily independent microservices.

I would avoid splitting every component into a separate deployment immediately. Payment orchestration, operation management, and outbox management can initially belong to one modular service.

The ledger should have a clearly defined ownership boundary. Whether it needs a separate database depends on regulatory, operational, and isolation requirements.

### **2. Architecture and service boundaries**

| Component            | Responsibility                              | Data ownership              |
| -------------------- | ------------------------------------------- | --------------------------- |
| Payment API          | Authentication, validation, idempotency     | API idempotency             |
| Payment Orchestrator | Payment lifecycle and business rules        | Payments, operations        |
| Provider Adapter     | Provider-specific API integration           | Provider request metadata   |
| Webhook Ingestion    | Signature verification, event normalization | Webhook inbox               |
| Ledger Service       | Double-entry accounting                     | Accounts, journals, entries |
| Reconciliation       | Detect and repair inconsistencies           | Reconciliation records      |
| Kafka                | Event distribution                          | Temporary event retention   |
| Outbox Relay         | Publish committed events                    | Outbox records              |

#### **Important ownership rule**

A provider adapter must never independently decide that a payment is financially complete.

It reports provider observations.

The payment orchestrator applies business rules and updates the internal payment state.

The ledger independently validates whether an accounting operation is valid before posting it.

This prevents provider-specific behavior from leaking into financial accounting logic.

#### **Single database or separate databases?**

My recommendation:

**Start with one PostgreSQL cluster, but use separate schemas and strict ownership boundaries.**

For example:

```
payment_schema
    payments
    payment_operations
    provider_requests
    outbox_events

webhook_schema
    webhook_inbox

ledger_schema
    ledger_accounts
    ledger_journals
    ledger_entries

reconciliation_schema
    reconciliation_runs
    reconciliation_items
```

This provides transactional consistency without introducing unnecessary distributed transactions.

A separate ledger database becomes attractive when independent security boundaries, regulatory requirements, independent scaling, or operational ownership justify the additional complexity.

If the ledger is separated, posting must become an asynchronous, idempotent workflow. You cannot assume atomic commits across both databases.

### **3. Payment state machine**

A payment state machine must distinguish between the payment's business lifecycle and individual financial operations.

A payment can have multiple captures and refunds. Therefore, a single status field cannot represent every financial fact.

#### **3.1 Payment-level states**

### **Payment lifecycle**

CREATED

PENDING

**Successful path**

AUTHORIZED

PARTIALLY_CAPTURED

CAPTURED

PARTIALLY_REFUNDED

REFUNDED

**Alternative paths**

FAILED

Definitive failure

VOIDED

Authorization released

UNKNOWN

Outcome requires investigation

Conceptual lifecycle. Actual transitions depend on provider capabilities and operation-level outcomes.

One correction to the simplified diagram: `VOIDED` is a terminal state for the authorization lifecycle, not necessarily for the entire payment record. A payment can also have multiple independent operations.

#### **3.2 Operation-level state machine**

Each capture, refund, authorization, and void should have its own state.

For example:

```
CREATED
   |
   v
PENDING
   |
   +------> SUCCEEDED
   |
   +------> FAILED
   |
   +------> UNKNOWN
                |
                v
          RECONCILING
             /    \
            v      v
       SUCCEEDED  FAILED
```

`UNKNOWN` is particularly important.

Consider:

1. Your service sends a capture request.
2. The provider processes it.
3. The provider's response is lost.
4. Your service times out.

The operation is not failed. It is unknown.

Automatically retrying with a new provider request could double-charge the customer.

#### **3.3 State transition rules**

| Current state      | Event                         | Result                         |
| ------------------ | ----------------------------- | ------------------------------ |
| CREATED            | Submit operation              | PENDING                        |
| PENDING            | Provider confirms success     | SUCCEEDED                      |
| PENDING            | Provider definitively rejects | FAILED                         |
| PENDING            | Network timeout               | UNKNOWN                        |
| UNKNOWN            | Provider confirms success     | SUCCEEDED                      |
| UNKNOWN            | Provider confirms rejection   | FAILED                         |
| AUTHORIZED         | Partial capture               | PARTIALLY_CAPTURED             |
| AUTHORIZED         | Full capture                  | CAPTURED                       |
| AUTHORIZED         | Successful void               | VOIDED                         |
| PARTIALLY_CAPTURED | Additional capture            | PARTIALLY_CAPTURED or CAPTURED |
| CAPTURED           | Partial refund                | PARTIALLY_REFUNDED             |
| CAPTURED           | Full refund                   | REFUNDED                       |
| PARTIALLY_REFUNDED | Additional refund             | PARTIALLY_REFUNDED or REFUNDED |

Do not permit a late `AUTHORIZED` event to overwrite a payment that has already been captured.

#### **3.4 Store facts, derive status**

I prefer storing operation records as immutable or carefully controlled lifecycle records and deriving aggregate financial status from their confirmed outcomes.

For example:

```
Payment: $100

Authorization:
  $100 SUCCEEDED

Capture #1:
  $40 SUCCEEDED

Capture #2:
  $60 SUCCEEDED

Refund #1:
  $20 SUCCEEDED

Net captured: $100
Net refunded: $20
Remaining captured value: $80
```

The aggregate status is `PARTIALLY_REFUNDED`.

The aggregate is a projection. The individual operations are the underlying financial facts.

This distinction makes event replay and reconciliation much safer.

### **4. Idempotency across the entire system**

This is one of the most important parts of the design.

Idempotency must be implemented independently at each boundary.

A single HTTP idempotency key does not solve distributed idempotency.

#### **4.1 API idempotency**

The API accepts:

```
POST /payments/{payment_id}/captures
Idempotency-Key: capture-12345
```

Store:

```
CREATE TABLE idempotency_records (
    tenant_id UUID NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    request_hash VARCHAR(64) NOT NULL,
    resource_id UUID,
    response_status INTEGER,
    response_body JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,

    PRIMARY KEY (tenant_id, idempotency_key)
);
```

Behavior:

| Situation                    | Behavior                                           |
| ---------------------------- | -------------------------------------------------- |
| First request                | Execute operation                                  |
| Same key, same payload       | Return original result or current operation status |
| Same key, different payload  | HTTP 409 Conflict                                  |
| Concurrent requests          | One request wins unique constraint                 |
| Previously completed request | Return stored response                             |

Do not simply return a cached `PENDING` response forever. Return the operation's current status or the original response according to the API contract.

The request hash should cover canonicalized business parameters, including amount, currency, payment ID, and operation type.

#### **4.2 Internal operation idempotency**

Every financial operation receives a permanent internal ID.

```
payment_id
operation_id
tenant_id
operation_type
amount
currency
status
provider_reference
```

Use:

```
UNIQUE (tenant_id, operation_id)
```

A capture operation should never be recreated simply because its HTTP request timed out.

#### **4.3 Provider idempotency**

Generate a deterministic provider key from the internal operation ID.

For example:

```
provider-idempotency-key =
    SHA256(tenant_id + operation_id + provider_account_id)
```

Persist this key before making the provider request.

Every retry of the same logical operation uses the same key.

**Critical limitation:** The provider retains keys for only 48 hours.

After 48 hours, blindly resubmitting the operation is unsafe.

The recovery process must first query the provider using the merchant reference or provider transaction reference.

If the provider cannot reliably establish whether the operation succeeded, escalate it for manual reconciliation. Do not generate a new key and retry automatically.

#### **4.4 Kafka idempotency**

Use a transactional outbox.

```
BEGIN;

UPDATE payment_operations
SET status = 'SUCCEEDED'
WHERE id = :operation_id;

INSERT INTO outbox_events (
    event_id,
    aggregate_id,
    event_type,
    payload,
    created_at
)
VALUES (
    :event_id,
    :payment_id,
    'PaymentCaptured',
    :payload,
    NOW()
);

COMMIT;
```

An outbox relay publishes committed records to Kafka.

If the relay crashes after publishing but before marking the event as published, it may publish again.

That is acceptable.

Consumers must be idempotent.

#### **4.5 Consumer idempotency**

Use an inbox table:

```
CREATE TABLE processed_events (
    consumer_name VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (consumer_name, event_id)
);
```

Process the event and record its ID in the same database transaction.

```
BEGIN;

INSERT INTO processed_events (
    consumer_name,
    event_id
)
VALUES (
    'ledger-consumer',
    :event_id
)
ON CONFLICT DO NOTHING
RETURNING event_id;
```

If no row is returned, the event has already been processed.

Otherwise, process the financial operation and commit.

```
COMMIT;
```

The database transaction must include both the inbox insertion and the business effect.

Otherwise, you create another crash window.

#### **4.6 Idempotency summary**

| Boundary           | Mechanism                                     |
| ------------------ | --------------------------------------------- |
| Client → API       | Tenant-scoped idempotency key                 |
| API → Database     | Unique constraints and transactions           |
| Database → Kafka   | Transactional outbox                          |
| Kafka → Consumer   | Transactional inbox                           |
| Consumer → Ledger  | Unique journal business reference             |
| Adapter → Provider | Stable provider idempotency key               |
| Provider → Webhook | Provider event ID + merchant account          |
| Reconciliation     | Stable operation ID and idempotent correction |

### **5. The hardest failure: provider succeeded, application crashed**

Consider the challenge's scenario:

```
1. Payment operation created.
2. Provider request sent.
3. Provider captures $100.
4. Provider returns success.
5. Application crashes.
6. Database still says PENDING.
7. Client retries.
```

The correct design does not depend on receiving the original HTTP response.

#### **Recovery workflow**

1. **Persist intent before external execution**

   Create the capture operation with `PENDING` status and its stable provider idempotency key.
2. **Send the provider request**

   The adapter uses the persisted key.
3. **Persist provider confirmation**

   If the application crashes, the operation remains `PENDING` or becomes `UNKNOWN` after timeout detection.
4. **Recover using the same operation ID**

   A background worker examines stale operations.
5. **Query provider status**

   Search by provider idempotency key, merchant reference, or transaction reference.
6. **Resolve the operation**

   If successful, update the operation and create the appropriate outbox event in one transaction.
7. **Post ledger entries**

   The ledger consumes the event idempotently.

The original client retry resolves to the same internal operation.

#### **Important design decision**

I would not hold a database transaction open while calling the provider.

Bad:

```
with transaction.atomic():    payment = Payment.objects.select_for_update().get(...)    provider.capture(...)    payment.status = "CAPTURED"    payment.save()
```

This causes long-lived locks and creates difficult failure behavior.

Better:

```
Transaction 1:
    Create operation
    Persist provider request intent
    Commit

Network:
    Call provider

Transaction 2:
    Record provider result
    Update operation
    Insert outbox event
    Commit
```

The gap between these transactions is intentional. It is handled by the recovery worker.

### **6. Kafka design: ordering and concurrency**

#### **6.1 Partitioning strategy**

I would partition payment lifecycle events by:

```
key = tenant_id + ":" + payment_id
```

This provides ordering for events belonging to the same payment within a Kafka topic.

However, there is an important limitation:

Kafka ordering is guaranteed only within a partition, not across different topics.

If payment events and ledger events are published to separate topics, you cannot assume a global ordering between them.

#### **6.2 Event envelope**

Use a standardized envelope:

```
{
  "event_id": "uuid",
  "event_type": "PaymentCaptured",
  "schema_version": 1,
  "tenant_id": "uuid",
  "payment_id": "uuid",
  "operation_id": "uuid",
  "aggregate_version": 12,
  "occurred_at": "2026-11-27T10:00:00Z",
  "published_at": "2026-11-27T10:00:01Z",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "payload": {
    "amount": "100.00",
    "currency": "USD"
  }
}
```

Use decimal strings for monetary values in event payloads. Never use floating-point numbers for financial amounts.

#### **6.3 Aggregate versioning**

Each payment maintains an aggregate version.

```
UPDATE payments
SET version = version + 1,
    updated_at = NOW()
WHERE id = :payment_id
  AND version = :expected_version;
```

If zero rows are updated, another transaction modified the payment.

The application must reload and reevaluate the transition.

This is optimistic concurrency control.

It is particularly useful when event processing involves competing updates.

#### **6.4 Out-of-order events**

Consider:

```
Event A: AUTHORIZED
Event B: CAPTURED
```

Actual provider sequence:

```
AUTHORIZED -> CAPTURED
```

Delivered webhook sequence:

```
CAPTURED -> AUTHORIZED
```

Do not blindly apply events based on arrival order.

Instead:

1. Persist every provider event.
2. Deduplicate using provider account ID and provider event ID.
3. Normalize the event.
4. Identify the corresponding internal operation.
5. Validate its transition against current financial facts.
6. Query provider state if necessary.
7. Update the projection only when justified.

A later-arriving authorization event must not undo a confirmed capture.

#### **6.5 Kafka consumer configuration**

I would use:

- Explicit consumer groups per business function.
- Manual offset commits after successful database commits.
- Bounded retries with exponential backoff.
- Dead-letter topics for permanently unprocessable messages.
- Schema compatibility checks.
- Consumer lag alerts.
- Bounded concurrency per payment partition.

A DLQ is not a mechanism for silently discarding financial events. It requires alerting, ownership, replay tooling, and operational SLAs.

### **7. PostgreSQL data model**

The schema should distinguish payment aggregates, individual operations, provider observations, and accounting records.

#### **7.1 Payments**

```
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,

    currency CHAR(3) NOT NULL,
    requested_amount NUMERIC(20, 4) NOT NULL,

    authorized_amount NUMERIC(20, 4) NOT NULL DEFAULT 0,
    captured_amount NUMERIC(20, 4) NOT NULL DEFAULT 0,
    refunded_amount NUMERIC(20, 4) NOT NULL DEFAULT 0,

    status VARCHAR(32) NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (requested_amount > 0),
    CHECK (authorized_amount >= 0),
    CHECK (captured_amount >= 0),
    CHECK (refunded_amount >= 0)
);

CREATE INDEX idx_payments_tenant_created
ON payments (tenant_id, created_at DESC);
```

The amounts are projections of confirmed operations, not independently trusted financial facts.

#### **7.2 Payment operations**

```
CREATE TABLE payment_operations (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    payment_id UUID NOT NULL REFERENCES payments(id),

    operation_type VARCHAR(32) NOT NULL,
    amount NUMERIC(20, 4) NOT NULL,
    currency CHAR(3) NOT NULL,

    status VARCHAR(32) NOT NULL,

    provider_id VARCHAR(64) NOT NULL,
    provider_reference VARCHAR(255),
    provider_idempotency_key VARCHAR(255) NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (amount > 0),

    UNIQUE (provider_id, provider_idempotency_key)
);

CREATE INDEX idx_operations_payment
ON payment_operations (tenant_id, payment_id);
```

In production, provider idempotency key uniqueness should include the provider's actual account or merchant scope, not merely the provider name.

#### **7.3 Provider webhook inbox**

```
CREATE TABLE provider_events (
    id UUID PRIMARY KEY,
    provider_id VARCHAR(64) NOT NULL,
    provider_account_id VARCHAR(128) NOT NULL,
    provider_event_id VARCHAR(255) NOT NULL,

    payload JSONB NOT NULL,
    processing_status VARCHAR(32) NOT NULL,

    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ,

    UNIQUE (
        provider_id,
        provider_account_id,
        provider_event_id
    )
);
```

Store the raw webhook payload securely. Encrypt sensitive data, enforce retention policies, and avoid logging payment credentials.

#### **7.4 Ledger schema**

```
CREATE TABLE ledger_accounts (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    account_code VARCHAR(64) NOT NULL,
    currency CHAR(3) NOT NULL,

    UNIQUE (tenant_id, account_code, currency)
);

CREATE TABLE ledger_journals (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,

    business_reference VARCHAR(255) NOT NULL,
    operation_type VARCHAR(32) NOT NULL,
    currency CHAR(3) NOT NULL,

    status VARCHAR(16) NOT NULL,
    posted_at TIMESTAMPTZ,

    UNIQUE (tenant_id, business_reference)
);

CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,
    journal_id UUID NOT NULL REFERENCES ledger_journals(id),
    account_id UUID NOT NULL REFERENCES ledger_accounts(id),

    debit NUMERIC(20, 4) NOT NULL DEFAULT 0,
    credit NUMERIC(20, 4) NOT NULL DEFAULT 0,

    CHECK (debit >= 0),
    CHECK (credit >= 0),
    CHECK (
        (debit > 0 AND credit = 0)
        OR
        (credit > 0 AND debit = 0)
    )
);
```

The application must enforce that:

\\[ \sum Debit = \sum Credit \\]

for each journal and currency.

PostgreSQL does not support a normal `CHECK` constraint that validates a sum across multiple rows. Enforce this through a posting transaction, a database procedure or deferred constraint trigger, and appropriate privileges.

#### **7.5 Immutability**

After a journal is posted:

- Do not update entries.
- Do not delete entries.
- Do not change the posted journal's financial content.
- Correct errors through compensating journals.

Enforce this with database permissions and triggers, not merely application conventions.

### **8. Double-entry ledger design**

A payment platform needs accounting semantics, not just a collection of payment statuses.

Consider a $100 capture.

Simplified journal:

| Account                | Debit    | Credit   |
| ---------------------- | -------- | -------- |
| Provider clearing      | $100     |          |
| Customer funds payable |          | $100     |
| **Total**              | **$100** | **$100** |

A $20 refund:

| Account                | Debit   | Credit  |
| ---------------------- | ------- | ------- |
| Customer funds payable | $20     |         |
| Provider clearing      |         | $20     |
| **Total**              | **$20** | **$20** |

The actual account mapping depends on whether the platform is a merchant of record, payment facilitator, or merely an orchestration provider.

Fees, settlement, reserves, chargebacks, and FX require additional journal types.

#### **Posting workflow**

```
PaymentCaptured event
        |
        v
Ledger consumer
        |
        v
BEGIN TRANSACTION
        |
        +--> Insert inbox event
        |
        +--> Check existing journal
        |
        +--> Validate operation
        |
        +--> Create journal
        |
        +--> Create debit entries
        |
        +--> Create credit entries
        |
        +--> Validate balance
        |
        +--> Mark journal POSTED
        |
        +--> Insert ledger outbox event
        |
        v
COMMIT
```

Use the internal operation ID as the journal business reference.

For example:

```
capture:operation-123
refund:operation-456
```

This makes duplicate ledger posting impossible when the unique constraint is enforced correctly.

#### **Important distinction**

A provider reporting a successful capture and the ledger recording the capture are separate facts.

The payment system should track both.

For example:

```
Provider operation: SUCCEEDED
Ledger posting: PENDING
```

This is a recoverable intermediate state, not an accounting inconsistency that should be hidden.

The system must alert if the ledger remains pending beyond its SLA.

### **9. Transaction boundaries**

This is where many apparently sound distributed architectures fail.

| Operation              | Database transaction                    | Network operation |
| ---------------------- | --------------------------------------- | ----------------- |
| Create payment         | Payment + idempotency + outbox          | None              |
| Initiate capture       | Operation + reserved amount + outbox    | None              |
| Send capture           | None held open                          | Provider API      |
| Record provider result | Operation + payment projection + outbox | None              |
| Ingest webhook         | Webhook inbox + outbox                  | None              |
| Post ledger            | Inbox + journal + entries + outbox      | None              |
| Publish event          | Outbox state update                     | Kafka             |

#### **Critical implementation detail: reserve amounts**

A simple check such as:

```
SELECT captured_amount
FROM payments
WHERE id = :payment_id;
```

followed by an application-side amount check is not sufficient.

Two concurrent capture requests can both observe the same remaining amount.

Use a reservation model.

For example:

```
Authorized amount:       $100
Captured amount:          $20
Capture reservations:     $30
Available to capture:     $50
```

The invariant is:

\\[ Captured + Reserved \le Authorized \\]

Perform the reservation under a row lock or an atomic conditional update.

```
BEGIN;

SELECT *
FROM payments
WHERE id = :payment_id
FOR UPDATE;

-- Validate available amount.
-- Create capture operation.
-- Reserve requested amount.

COMMIT;
```

When the provider confirms success, move the amount from reserved to captured.

When the provider definitively rejects the operation, release the reservation.

When the result is unknown, retain the reservation until reconciliation resolves it.

This prevents a second capture from consuming funds that may already have been captured externally.

### **10. Reconciliation architecture**

Reconciliation is not an optional administrative tool.

It is part of the core correctness model.

[White-Label Payment Gateway Vendor: How to Choose](https://images.openai.com/static-rsc-4/uXNzMyF7-7DoByccyJjgpmkiAbYrFVYyuEENC426cSzP99O5ICxkP5ibhhIwHMJZ0dlCPvWpQax6qbREys_LsDF2nwWJyrvEB24ROXLm3bBAH-bVmjBtIMfujmW2grjJ59leQIIbmg5IAs3OX0RLRtqo-twtjXxLw8iKNBj_XJI?purpose=inline)

[finon.tech](https://finon.tech/blog/white-label-payment-gateway-rfp-checklist?utm_source=chatgpt.com)

#### **10.1 Reconciliation dimensions**

I would implement three independent reconciliation processes.

| Process                   | Comparison                                                |
| ------------------------- | --------------------------------------------------------- |
| Operation reconciliation  | Internal operations vs provider transaction status        |
| Ledger reconciliation     | Confirmed payment operations vs ledger journals           |
| Settlement reconciliation | Internal clearing balances vs provider settlement reports |

These solve different problems.

A provider transaction can be successful while its ledger journal is missing. Both systems can be internally consistent while the settlement amount is wrong.

#### **10.2 Reconciliation workflow**

```
Scheduled reconciliation
        |
        v
Fetch provider transactions
        |
        v
Normalize provider records
        |
        v
Match using stable references
        |
        v
Compare internal records
        |
        +------ Match ------> Mark reconciled
        |
        +------ Mismatch ---> Create exception
                                  |
                                  v
                           Classify mismatch
                                  |
                     +------------+-----------+
                     |            |           |
                     v            v           v
                  Missing      Status      Amount
                  webhook      mismatch    mismatch
                     |            |           |
                     v            v           v
                  Replay       Query       Escalate
                  event        provider    investigate
```

#### **10.3 Safe automatic repair**

Examples:

- Provider confirms successful capture, internal operation is `UNKNOWN`: transition operation to `SUCCEEDED`.
- Operation is successful but ledger journal is absent: create the missing journal.
- Duplicate webhook: ignore the duplicate.
- Provider confirms definitive failure: resolve the operation and release its reservation.

Never automatically issue a new capture or refund merely because a ledger entry is missing.

A missing ledger entry is an accounting repair problem, not a reason to move money again.

#### **10.4 Reconciliation scheduling**

Suggested schedule:

| Job                                | Frequency           |
| ---------------------------------- | ------------------- |
| Stale unknown operations           | Every 1–5 minutes   |
| Missing webhook detection          | Every 5 minutes     |
| Recent payment reconciliation      | Every 15–30 minutes |
| Provider settlement reconciliation | Daily               |
| Historical audit                   | Daily or weekly     |

Apply provider-specific rate limits, exponential backoff, pagination, and bounded concurrency.

For historical records, use time-based partitions and incremental checkpoints.

### **11. Failure-injection scenario solutions**

The following table summarizes the expected behavior.

| ##  | Failure                                      | Correct recovery                                                                            |
| -- | -------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 1  | API crashes after payment commit             | Outbox relay and retry recover the operation.                                               |
| 2  | Provider captures but response times out     | Mark `UNKNOWN`; query provider using the original reference and idempotency key.            |
| 3  | Kafka acknowledgment lost                    | Republish outbox event; consumer inbox deduplicates.                                        |
| 4  | Consumer crashes after DB commit             | Kafka redelivers; inbox prevents duplicate effects.                                         |
| 5  | Two consumers process same capture           | Unique inbox key, journal key, and row-level locking prevent duplicate posting.             |
| 6  | REFUNDED arrives before CAPTURED             | Persist both; validate provider operation facts and reconcile. Never blindly regress state. |
| 7  | Webhook delivered 100 times                  | Provider account + event ID uniqueness constraint.                                          |
| 8  | Database primary fails during ledger posting | Transaction either commits or rolls back. Retry through Kafka.                              |
| 9  | Provider unavailable for 45 minutes          | Queue durable intents, apply backoff, circuit breaker, and preserve reservations.           |
| 10 | Provider capture absent from ledger          | Post a missing journal idempotently. Do not capture again.                                  |
| 11 | Old webhook after refund                     | Preserve event history; reconcile against current operation facts.                          |
| 12 | Schema changes during deployment             | Backward-compatible event evolution and staged consumer deployment.                         |

#### **Three particularly difficult cases**

**Case 5: Concurrent consumers**

Kafka normally assigns one partition to one consumer within a consumer group. However, application-level concurrency, retries, replay, and other consumer groups can still cause concurrent access.

Do not rely on Kafka partition ownership as your financial locking mechanism.

Use database uniqueness and transactional validation.

**Case 6: Refund before capture**

Do not reject the refund permanently just because the capture event has not arrived.

The capture may already exist at the provider.

Persist the refund observation and establish the underlying operation relationships. Query the provider and resolve the actual sequence.

**Case 12: Schema evolution**

Use Avro with a Schema Registry.

Apply:

1. Additive schema changes.
2. Optional fields with defaults.
3. Backward compatibility checks.
4. Consumer-first deployment where applicable.
5. Removal of old fields only after all consumers have migrated.

Never change the meaning of an existing field while retaining the same schema version.

### **12. Performance and scalability**

At 15,000 API requests/sec and 25,000 webhook events/sec, database design and workload isolation matter more than simply adding application replicas.

#### **12.1 PostgreSQL optimization**

My recommendations:

| Area                  | Recommendation                                   |
| --------------------- | ------------------------------------------------ |
| Connection management | PgBouncer or equivalent pooling                  |
| Transactions          | Keep transactions short                          |
| Locking               | Lock individual payment aggregates               |
| Reads                 | Read replicas for non-critical queries           |
| Indexes               | Composite indexes matching tenant-scoped queries |
| Large tables          | Time-based partitioning                          |
| Historical data       | Archive cold records                             |
| Writes                | Batch outbox publishing                          |
| Contention            | Avoid global counters and hot rows               |

For example:

```
CREATE INDEX CONCURRENTLY idx_payments_tenant_status_created
ON payments (
    tenant_id,
    status,
    created_at DESC
)
INCLUDE (
    id,
    requested_amount,
    currency
);
```

Use partial indexes for frequently queried subsets:

```
CREATE INDEX CONCURRENTLY idx_pending_operations
ON payment_operations (created_at)
WHERE status IN ('PENDING', 'UNKNOWN');
```

The partial index reduces the indexed dataset to operationally relevant records.

For three billion payment records, partitioning strategy must be designed around actual query patterns, retention, and maintenance operations. Avoid partitioning exclusively by tenant when there are 20,000 tenants; that creates excessive partition management overhead.

#### **12.2 Kafka optimization**

Use:

- Adequate partition counts based on measured throughput.
- Compression such as LZ4 or Zstandard.
- Bounded consumer concurrency.
- Separate topics for distinct workload classes.
- Independent consumer groups for payment, ledger, and notifications.
- Backpressure when downstream databases become overloaded.

Do not increase consumer concurrency indefinitely. PostgreSQL may become the bottleneck before Kafka does.

#### **12.3 Tenant isolation**

A high-volume tenant must not consume all available capacity.

Implement:

- Per-tenant API rate limits.
- Per-tenant concurrency limits.
- Fair scheduling for background work.
- Tenant-aware partition distribution.
- Separate priority queues for critical financial operations.

For database isolation, use PostgreSQL Row-Level Security as an additional safeguard.

```
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation
ON payments
USING (
    tenant_id = current_setting('app.tenant_id')::UUID
);
```

Set the tenant context transaction-locally. Be careful with pooled connections to prevent tenant context leakage.

### **13. Observability and incident management**

I would establish four observability dimensions.

#### **13.1 Business metrics**

| Metric                        | Purpose                       |
| ----------------------------- | ----------------------------- |
| Payment success rate          | Business health               |
| Capture success rate          | Financial operation health    |
| Refund success rate           | Refund processing health      |
| Unknown operation count       | Unresolved financial outcomes |
| Payment completion latency    | Customer experience           |
| Reconciliation mismatch count | Financial integrity           |

#### **13.2 Technical metrics**

| Metric                           | Purpose                       |
| -------------------------------- | ----------------------------- |
| API p95/p99 latency              | API performance               |
| PostgreSQL connection saturation | Database capacity             |
| Lock wait duration               | Contention                    |
| Kafka consumer lag               | Processing delay              |
| Outbox backlog                   | Publication health            |
| Webhook processing latency       | Provider event responsiveness |
| Provider timeout rate            | Integration health            |
| DLQ message count                | Processing failures           |

#### **13.3 Financial integrity alerts**

These should be high-priority alerts:

- Unbalanced journal detected.
- Capture exceeds authorization.
- Refund exceeds captured amount.
- Successful provider operation missing ledger journal.
- Duplicate journal business reference conflict.
- Unknown operation exceeds investigation SLA.
- Provider settlement mismatch.
- Unauthorized cross-tenant access attempt.

#### **13.4 Distributed tracing**

Use OpenTelemetry.

Propagate:

```
trace_id
correlation_id
causation_id
tenant_id
payment_id
operation_id
provider_reference
event_id
journal_id
```

Example:

```
HTTP Request
    |
    +-- Payment API
    |
    +-- Payment Orchestrator
    |
    +-- PostgreSQL transaction
    |
    +-- Outbox Relay
    |
    +-- Kafka
    |
    +-- Provider Adapter
    |
    +-- Provider API
    |
    +-- Webhook Ingestion
    |
    +-- Ledger Consumer
    |
    +-- Ledger PostgreSQL
```

Avoid putting high-cardinality identifiers such as payment IDs into Prometheus metric labels. Use logs and traces for individual transaction investigation.

### **14. Disaster recovery**

I would establish explicit recovery objectives.

| Component            | Proposed RPO                                                                | Proposed RTO       |
| -------------------- | --------------------------------------------------------------------------- | ------------------ |
| Payment database     | 0 for acknowledged committed transactions within synchronous HA scope       | 5 minutes          |
| Ledger database      | 0 within synchronous HA scope                                               | 5 minutes          |
| Kafka                | 0 for acknowledged durable records within configured replication guarantees | 15 minutes         |
| Provider integration | External dependency                                                         | Provider-dependent |

These are design targets, not automatic guarantees.

#### **PostgreSQL**

- Multi-AZ synchronous replication.
- Automated failover.
- Point-in-time recovery.
- Continuous backup.
- Regular restore testing.
- Independent ledger backup and audit procedures.

Be precise about RPO: asynchronous cross-region replication cannot guarantee zero data loss.

#### **Kafka**

- Replication factor 3.
- `acks=all`.
- Appropriate `min.insync.replicas`.
- Idempotent producers.
- Retention appropriate to replay requirements.
- Replicated schema registry metadata.

Kafka durability does not replace PostgreSQL durability. The outbox remains the source of truth for events that must eventually be published.

#### **Regional disaster**

For a regional failure:

1. Promote the secondary region according to the recovery plan.
2. Verify database consistency.
3. Restore or recover Kafka event streams.
4. Reconcile outstanding operations with payment providers.
5. Resume provider integrations.
6. Reconcile payments and ledger journals.
7. Restore normal customer traffic.

Do not assume that an external provider has the same regional failure state as your platform.

### **15. Engineering trade-offs**

A principal engineer must understand what the architecture costs.

| Decision                      | Benefit                                   | Cost                                          |
| ----------------------------- | ----------------------------------------- | --------------------------------------------- |
| Transactional outbox          | No lost committed domain events           | Additional database writes and relay          |
| Transactional inbox           | Duplicate processing protection           | Storage and transaction overhead              |
| Separate ledger ownership     | Strong accounting boundaries              | More operational complexity                   |
| PostgreSQL row locks          | Strong financial consistency              | Potential contention                          |
| Kafka partitioning by payment | Per-payment event ordering                | Partition skew                                |
| Reconciliation                | Recovery from distributed inconsistencies | Provider API load and operational cost        |
| Immutable ledger              | Auditable financial history               | Corrections require compensating entries      |
| Event-driven architecture     | Decoupling and scalability                | Eventual consistency and debugging complexity |

#### **Three things I would avoid**

**1. Claiming exactly-once processing solves financial correctness.**

Kafka exactly-once semantics do not make an external provider API transactional.

**2. Using distributed transactions across PostgreSQL, Kafka, and payment providers.**

The provider does not participate in your database transaction. Design for recoverable partial completion instead.

**3. Treating payment status as the source of financial truth.**

A status is a projection. Financial operations and posted ledger entries are the auditable facts.

### **16. Recommended implementation roadmap**

I would implement this incrementally rather than build every component simultaneously.

1. **Phase 1 — Financial foundation**
   - Define payment and operation state machines.
   - Implement idempotent API commands.
   - Introduce amount reservations.
   - Implement the double-entry ledger.
   - Enforce database constraints.
2. **Phase 2 — Reliable asynchronous processing**
   - Introduce transactional outbox.
   - Introduce Kafka event contracts.
   - Implement transactional inbox.
   - Implement consumer retry and DLQ workflows.
3. **Phase 3 — Provider integration**
   - Implement provider adapters.
   - Persist provider request intent.
   - Implement stable provider idempotency.
   - Introduce webhook ingestion and deduplication.
   - Implement unknown-outcome recovery.
4. **Phase 4 — Financial reconciliation**
   - Implement operation reconciliation.
   - Implement ledger reconciliation.
   - Implement settlement reconciliation.
   - Add manual investigation and repair workflows.
5. **Phase 5 — Production hardening**
   - Load testing.
   - Failure injection.
   - Database failover testing.
   - Kafka replay testing.
   - Disaster recovery exercises.
   - Financial audit verification.

### **17. Final architecture principles**

These are the principles I would enforce during architecture reviews.

1. **Persist intent before executing external side effects.**
2. **Never assume a timeout means an operation failed.**
3. **Every financial operation has a stable internal identity.**
4. **Every distributed boundary has an idempotency strategy.**
5. **Use transactional outbox and inbox patterns.**
6. **Use database constraints as correctness mechanisms, not just validation.**
7. **Reserve authorized amounts before executing concurrent captures.**
8. **Treat provider events as observations, not unquestionable commands.**
9. **Never modify posted ledger entries.**
10. **Reconciliation is a permanent part of the architecture.**

#### **My assessment of the most important insight**

### **The difference between reliable systems and financially correct systems**

A reliable distributed system delivers messages and recovers from failures.

A financially correct distributed system additionally guarantees that every confirmed movement of money has exactly one corresponding accounting effect, that no unauthorized movement is created, and that discrepancies can be detected and repaired.

**Delivery guarantees are transport properties. Financial correctness is a business invariant.**

That distinction is the foundation of a production-grade payment architecture.
