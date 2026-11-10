\# OrderFlow — Architecture



Version: 0.1

Status: Draft

Related Issue: #3



> All architectural decisions in this document are proposals.

> Nothing described here is implemented yet.



\## 1. Services and Responsibilities



| Service | Responsibility | Owned data |

|---|---|---|

| Order Service | Exposes the REST API. Manages the order lifecycle and orchestrates the order saga through events. | Orders, order items, cancellation reasons |

| Inventory Service | Validates availability, reserves and releases stock. | Products, stock levels, reservations |

| Payment Service | Simulates payment approval or rejection. | Payments |

| Notification Service | Reacts to order status events and records notifications. | Notification log |



\### 1.1 Boundaries



\- Only the Order Service exposes a REST API to the API Consumer.

\- Each service owns its own PostgreSQL database. No service reads or writes another service's database.

\- The main business workflow uses asynchronous communication through Kafka. No synchronous calls between services.

\- Each service is deployed and started independently.

### 1.2 Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Saga style | Orchestration, coordinated by the Order Service | The order state machine already tracks each step, so a stuck order is visible and diagnosable. |
| Notifications | Only on `OrderConfirmed` and `OrderCancelled` | Intermediate states are internal and would create noise. |

## 2. Production Principles

These are design goals. None of them is implemented yet.

### 2.1 Reliability

- **Transactional Outbox:** a service writes its state change and the event to publish in the same database transaction. A separate publisher sends outbox rows to Kafka. This avoids the dual-write problem (database updated but event lost, or the reverse).
- **Idempotent consumers:** each service stores processed `eventId` values in its own database, in the same transaction as the state change. Reprocessing a duplicate has no effect.
- **At-least-once delivery:** offsets are committed only after the database transaction succeeds.
- **Producer settings:** `acks=all` and idempotent producer enabled.
- **Retry and DLT:** bounded retries with backoff, then the message goes to a dead-letter topic (`<topic>.DLT`) with the error recorded. Errors that will never succeed (invalid payload) skip retries and go straight to the DLT.

### 2.2 Ordering

- Every message is keyed by `orderId`, so all events of one order go to the same partition and are processed in order.
- Consumers must tolerate out-of-order and duplicated events and reject invalid state transitions.

### 2.3 Compensation failure

- If releasing inventory fails, the order stays in `CANCELLATION_PENDING`. It is never marked `CANCELLED` until `InventoryReleased` is received.
- Release is retried. After retries are exhausted, the message goes to the DLT and an alert is raised for manual intervention.
- Releasing the same reservation twice has no effect (BR07).

### 2.4 Stuck orders

- A scheduled reconciliation job in the Order Service detects orders that stay in `PENDING`, `INVENTORY_RESERVED` or `CANCELLATION_PENDING` longer than a configured timeout.
- Detected orders are logged, counted in a metric and trigger an alert. The recovery policy will be defined later.

### 2.5 Observability

- Every event carries `correlationId` (the order flow) and `causationId` (the event that caused it).
- Structured logs include `orderId`, `eventId` and `correlationId`.
- Metrics: consumer lag, DLT message count, orders per status, saga duration.

### 2.6 Environments

- Local: single Kafka broker, replication factor 1.
- Production target: 3 brokers, replication factor 3, `min.insync.replicas=2`.

## 3. Events and Topics

### 3.1 Event envelope

Every event uses the same envelope:

```json
{
  "eventId": "uuid",
  "eventType": "InventoryReserved",
  "schemaVersion": 1,
  "occurredAt": "2026-01-01T12:00:00Z",
  "orderId": "uuid",
  "correlationId": "uuid",
  "causationId": "uuid",
  "payload": {}
}
```

Changes to `payload` must be backward compatible within the same `schemaVersion`.

### 3.2 Topics

| Topic | Producer | Consumer | Messages |
|---|---|---|---|
| `order.events` | Order Service | Notification Service | `OrderCreated`, `OrderConfirmed`, `OrderCancelled` |
| `inventory.commands` | Order Service | Inventory Service | `InventoryReservationRequested`, `InventoryReleaseRequested` |
| `inventory.events` | Inventory Service | Order Service | `InventoryReserved`, `InventoryReservationFailed`, `InventoryReleased` |
| `payment.commands` | Order Service | Payment Service | `PaymentRequested` |
| `payment.events` | Payment Service | Order Service | `PaymentApproved`, `PaymentRejected` |

Each topic has a matching `<topic>.DLT`.

### 3.3 Saga flows

**Happy path**

1. `POST /orders` creates the order as `PENDING` and writes `InventoryReservationRequested` to the outbox.
2. Inventory Service reserves stock and publishes `InventoryReserved`.
3. Order Service moves the order to `INVENTORY_RESERVED` and publishes `PaymentRequested`.
4. Payment Service publishes `PaymentApproved`.
5. Order Service moves the order to `CONFIRMED` and publishes `OrderConfirmed`.

**Inventory reservation fails**

1. Inventory Service publishes `InventoryReservationFailed`.
2. Order Service moves the order directly to `CANCELLED` with the reason, because there is nothing to compensate, and publishes `OrderCancelled`.

**Payment rejected**

1. Payment Service publishes `PaymentRejected`.
2. Order Service moves the order to `CANCELLATION_PENDING` and publishes `InventoryReleaseRequested`.
3. Inventory Service releases the reservation and publishes `InventoryReleased`.
4. Order Service moves the order to

