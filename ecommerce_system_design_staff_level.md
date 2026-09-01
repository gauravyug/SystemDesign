# Design an E-commerce System — Staff / Senior Staff Level

Think of a simplified Amazon: users browse products, search, add items to cart, place orders, pay, track orders, and sellers/admins manage inventory.

---

## 1. Requirements

### Functional requirements

Core flows:

- Browse product catalog
- Search products
- View product details
- Add/remove/update items in cart
- Check inventory availability
- Checkout
- Apply coupons/promotions
- Make payment
- Create order
- Reserve/decrement inventory
- Track order status
- Cancel/refund orders
- Send notifications
- Seller/admin can update products, pricing, and inventory

At Staff level, explicitly separate:

### Browsing path
- Extremely read-heavy
- Eventual consistency is generally acceptable

### Checkout path
- Lower traffic
- Correctness is much more important
- Inventory, order, and payment need stronger guarantees

That distinction drives most architecture decisions.

---

## 2. Non-functional requirements

### Availability
- Browse/search: 99.99%+
- Checkout/order/payment: 99.9%+ but correctness over availability in certain failure cases

### Latency targets
- Product page: p99 < 300 ms
- Search: p99 < 500 ms
- Add to cart: p99 < 200 ms
- Checkout initiation: < 1 second excluding payment-provider latency

### Consistency
- Product descriptions: eventual consistency
- Search index: eventual consistency
- Inventory during checkout: strong enough to prevent overselling
- Orders: strongly consistent
- Payments: durable and idempotent

### Durability
- Never silently lose an acknowledged order/payment state.

### Scalability
- Horizontal scaling
- Support flash sales and highly skewed product popularity

---

## 3. Scale assumptions

Assume:

```text
100M registered users
10M DAU

Products:              100M
Product page requests: 100K RPS peak
Search:                50K RPS peak
Cart operations:       20K RPS
Checkout:               5K RPS
Orders:                  2K/sec peak
```

Important Staff-level observation:

> We should not design every subsystem for the same consistency or scalability model.

Product/search traffic is several orders of magnitude larger than checkout.

---

## 4. High-level architecture

```text
                        +----------------+
                        | Web / Mobile   |
                        +-------+--------+
                                |
                         CDN / Edge Cache
                                |
                         API Gateway / BFF
                                |
       +------------------------+-------------------------+
       |             |             |                    |
       v             v             v                    v
+-------------+ +----------+ +------------+      +-------------+
| Catalog     | | Search   | | Cart       |      | User        |
| Service     | | Service  | | Service    |      | Service     |
+------+------+ +----+-----+ +------+-----+      +-------------+
       |             |              |
       v             v              v
 Product DB      Search Index      Redis / DB


                         CHECKOUT PATH

                            Client
                              |
                        Checkout Service
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
       Pricing/Promo     Inventory        Order Service
          Service         Service               |
                                               DB
             |                |
             +--------+-------+
                      |
                Payment Service
                      |
             Payment Provider(s)
           Stripe / Adyen / Banks

                      |
                 Event Bus
                 Kafka/Pulsar
                      |
       +--------------+-------------------+
       |              |                   |
       v              v                   v
 Fulfillment     Notification       Analytics
 Service         Service            / Data Lake
```

---

## 5. Catalog Service

The catalog contains:

```text
Product
-------
product_id
seller_id
title
description
category
attributes
images
brand
status
```

Important separation:

**Catalog != price != inventory**

A common weak design puts all three in the Product table.

That causes problems because:

- Catalog changes infrequently
- Prices change frequently
- Inventory changes extremely frequently

They have different scaling and consistency requirements.

So use:

```text
Catalog Service
Pricing Service
Inventory Service
```

as separate ownership boundaries.

For catalog storage:

- Relational DB if schema is fairly structured
- DynamoDB/Cassandra/document database if huge flexible product attributes

Product images go into object storage:

```text
S3 / Blob Storage
        |
       CDN
```

---

## 6. Search architecture

Never execute customer search directly against the primary product database.

Use:

```text
Catalog DB
   |
 CatalogUpdated event
   |
 Kafka
   |
 Indexer
   |
Elasticsearch / OpenSearch
```

Search becomes intentionally eventually consistent.

Example:

```text
Seller changes:

iPhone 17
₹79,999 -> ₹74,999
```

There can be a few seconds before search reflects the change.

But during checkout we **never trust the search index price**.

We re-fetch current price from Pricing Service.

That's an important interview point.

---

## 7. Cart Service

Cart is high throughput but doesn't require full transactional durability.

Data:

```text
cart_id
user_id

CartItem
--------
product_id
quantity
selected
added_at
```

Use something like:

```text
Redis
```

with persistent backing or asynchronous DB persistence depending on durability requirements.

For anonymous customers:

```text
cart_id -> cookie/session
```

For authenticated users:

```text
user_id -> cart
```

We should **not reserve inventory when something is added to cart**.

Why?

```text
10 laptops available
1000 users add laptop to cart
```

If cart reserves stock, inventory becomes unusable without a purchase.

Reservation should happen during checkout.

---

## 8. Checkout orchestration

Suppose user buys:

```text
Laptop x1
Mouse  x2
```

Checkout must coordinate:

```text
pricing
promotions
inventory
payment
order
```

But these services use separate databases.

Therefore we cannot realistically use a traditional distributed ACID transaction.

Instead use a **Saga / workflow**.

A durable workflow orchestrator could be:

```text
Temporal
Cadence
or an internal workflow engine
```

Conceptually:

```text
START CHECKOUT

1. Validate cart
2. Calculate authoritative price
3. Apply promotion
4. Reserve inventory
5. Create order = PENDING_PAYMENT
6. Authorize payment
7. Confirm order
8. Commit inventory reservation

FAILURE:
payment fails
   |
release inventory
   |
mark order PAYMENT_FAILED
```

---

## 9. Why inventory reservation instead of direct decrement?

Suppose only one PS5 remains.

Two customers concurrently try:

```text
Customer A -> buy 1
Customer B -> buy 1
```

Naive implementation:

```text
SELECT quantity
quantity = 1

A reads 1
B reads 1

both decrement
```

Overselling.

Instead maintain:

```text
total_quantity
reserved_quantity
available_quantity
```

where:

```text
available = total - reserved
```

Reservation operation must be atomic.

For SQL:

```sql
UPDATE inventory
SET reserved = reserved + 1
WHERE product_id = ?
AND total - reserved >= 1;
```

Then check:

```text
rows_updated == 1
```

Only one concurrent customer wins.

---

## 10. Reservation expiration

A customer can reserve inventory and then disappear during payment.

Therefore reservation has:

```text
reservation_id
product_id
quantity
expires_at
status
```

Example:

```text
reservation TTL = 10 minutes
```

A background worker releases expired reservations.

But don't rely solely on TTL cleanup.

Checkout workflow should explicitly:

```text
release reservation
```

on failure.

The cleanup worker is the safety net.

---

## 11. Order Service

Order is the user's durable business record.

Possible state machine:

```text
CREATED
   |
   v
PENDING_PAYMENT
   |
   +------ payment failed -----> PAYMENT_FAILED
   |
 payment success
   v
CONFIRMED
   |
   v
PROCESSING
   |
   v
SHIPPED
   |
   v
DELIVERED
```

Cancellation path:

```text
CONFIRMED -> CANCELLED
```

Refund:

```text
CANCELLED
   |
REFUND_PENDING
   |
REFUNDED
```

Do not model this as random strings updated by every service.

Order Service owns the state machine.

---

## 12. Order database

For the transactional order system, prefer relational storage.

Example:

```text
orders
------
order_id PK
user_id
status
currency
subtotal
discount
tax
shipping
total
version
created_at

order_items
-----------
order_id
product_id
seller_id
quantity
unit_price
discount
tax

order_status_history
--------------------
order_id
old_status
new_status
timestamp
reason
```

Why store `unit_price` in order items rather than querying Pricing Service later?

Because the order is a **historical business record**.

If product price changes tomorrow, yesterday's order must remain unchanged.

---

## 13. Payment Service

Payment should have a separate entity from Order.

```text
Payment
-------
payment_id
order_id
amount
currency
provider
provider_payment_id
status
idempotency_key
created_at
```

Payment status:

```text
CREATED
AUTHORIZED
CAPTURED
FAILED
REFUND_PENDING
REFUNDED
```

Important:

```text
Order != Payment
```

One order may have:

- Payment retry
- Multiple payment attempts
- Partial refund
- Multiple refunds

So use:

```text
Order
   |
   +---- PaymentAttempt 1 FAILED
   +---- PaymentAttempt 2 SUCCESS
```

---

## 14. Idempotency

Suppose:

```text
POST /checkout
```

succeeds, but the client's network times out.

Client retries.

Without idempotency:

```text
Order #123 created
Order #124 created
```

Possibly double charge.

Client sends:

```http
Idempotency-Key: abc123
```

Server stores:

```text
abc123 -> order_id 123
```

Retry:

```text
same key
      |
return existing response
```

This must apply at multiple layers:

```text
Checkout API
Payment Service
Inventory reservation
Event consumers
```

Staff-level point:

> Exactly-once delivery is usually unrealistic. Build effectively-once business behavior using at-least-once delivery plus idempotency.

---

## 15. External payment-provider failure

Suppose:

```text
Payment Service
    |
    | charge ₹10,000
    v
Provider

Provider processes payment
but response is lost.
```

Our system cannot safely say:

```text
FAILED
```

because the customer may have been charged.

Status becomes:

```text
PAYMENT_PENDING / UNKNOWN
```

Then resolve through:

```text
provider API query
webhook
reconciliation
```

Never blindly retry a payment when outcome is unknown unless the provider's API itself supports idempotency.

---

## 16. Webhooks

Payment provider sends:

```text
POST /payment-webhook
```

Example:

```text
payment_id = xyz
status = CAPTURED
```

Webhook processing must:

- Authenticate signature
- Be idempotent
- Persist before acknowledging
- Handle out-of-order events

Example:

```text
CAPTURED arrives
then delayed AUTHORIZED arrives
```

We shouldn't move backward:

```text
CAPTURED -> AUTHORIZED
```

The state machine prevents invalid transition.

---

## 17. Transactional Outbox

Classic distributed systems problem:

```text
Order DB commit succeeds
Kafka publish fails
```

Order exists, but downstream services never know.

Don't do:

```text
DB commit
then
Kafka publish
```

independently.

Use transactional outbox:

```text
DB transaction:

INSERT order
INSERT outbox_event

COMMIT
```

Then:

```text
Outbox Publisher
      |
     Kafka
```

Both business state and event intent are committed atomically.

Consumers remain idempotent because duplicate events are possible.

---

## 18. Event architecture

Useful domain events:

```text
OrderCreated
InventoryReserved
PaymentAuthorized
PaymentCaptured
OrderConfirmed
OrderCancelled
InventoryReleased
ShipmentCreated
OrderDelivered
RefundInitiated
RefundCompleted
```

Kafka topics could be partitioned by:

```text
order_id
```

That gives ordering for events belonging to an order.

Not global ordering.

We don't need global ordering and shouldn't pay its scalability cost.

---

## 19. Flash sale / hot key problem

Suppose Taylor Swift merchandise has:

```text
100 items
5 million customers
```

Every request attacking one inventory row will destroy the database.

We need special handling.

Potential architecture:

```text
API
 |
Rate Limiter
 |
Admission Queue
 |
Inventory allocator
 |
Order creation
```

For extreme flash sales:

```text
500 remaining units
```

We can pre-create inventory tokens:

```text
token001
token002
...
token500
```

Atomic token acquisition determines winners.

Only winning requests continue to checkout.

This reduces DB contention drastically.

---

## 20. Caching strategy

Different data has different cacheability.

```text
Product description -> CDN / Redis
Images              -> CDN
Category pages       -> CDN/cache
Search               -> search engine cache
Price                -> short TTL
Inventory            -> cache cautiously
Cart                 -> Redis
```

Important:

**Never make final checkout decision using cached inventory.**

Cache may display:

```text
"Only 3 left"
```

but the authoritative reservation operation decides whether purchase succeeds.

---

## 21. Database partitioning

At large scale, Orders can't live indefinitely on one DB.

Partition by:

```text
user_id
```

if dominant pattern is:

```text
show me my orders
```

or potentially:

```text
order_id hash
```

depending on workload.

We need carefully designed secondary access paths for:

```text
seller orders
fulfillment center orders
support lookup
```

Don't add arbitrary global secondary indexes to the write path.

Push derived views through events into dedicated read stores.

---

## 22. CQRS where useful

The write-side order DB optimizes correctness.

But customer-facing order history may need:

```text
product thumbnail
seller name
delivery status
payment status
tracking status
```

Instead of joining six microservice DBs at request time, create an:

```text
Order View
```

updated asynchronously:

```text
Kafka
   |
OrderProjectionConsumer
   |
OrderReadDB
```

Then:

```text
GET /users/{id}/orders
```

reads the projection.

This is a practical use of CQRS, not CQRS for architecture-fashion reasons.

---

## 23. Multi-region design

This is where Staff interviews often get interesting.

Browsing can be:

```text
Active-Active
```

globally.

```text
US region
EU region
India region
```

Catalog and search can replicate asynchronously.

Checkout is harder.

For each order, one region should be authoritative.

For example:

```text
user/order -> home region
```

Order writes remain single-region strongly consistent.

Why not active-active DB writes everywhere?

Inventory creates a global concurrency problem:

```text
US believes stock = 1
India believes stock = 1
```

Both could sell the final item.

Better strategies include:

```text
inventory ownership by region
```

or:

```text
regional inventory pools
```

Example:

```text
US:    40
EU:    30
India: 30
```

Then redistribute asynchronously when needed.

This is often much better operationally than global consensus on every purchase.

---

## 24. Disaster recovery

Define explicit targets:

```text
Catalog:
RPO: minutes
RTO: minutes

Orders:
RPO: ~0
RTO: < 5 minutes

Payments:
RPO: ~0
RTO: < 5 minutes
```

For orders/payments:

```text
multi-AZ synchronous replication
+
cross-region replication
```

But if cross-region replication is asynchronous, acknowledge the tradeoff:

A regional disaster can lose acknowledged writes.

If business requires RPO=0 even across regions, synchronous cross-region replication is needed, which increases write latency.

This is exactly the kind of trade-off to discuss at Staff level.

---

## 25. What if checkout crashes midway?

Suppose workflow reaches:

```text
Inventory reserved ✅
Order created ✅
Payment succeeds ✅
```

then Checkout Service crashes before marking order confirmed.

A weak solution depends on the process surviving.

A robust solution uses durable workflow state:

```text
Workflow ID = checkout/order ID
```

On restart:

```text
resume from last durable step
```

And all operations are idempotent.

Calling:

```text
capturePayment(payment123)
```

again should return existing result instead of charging twice.

---

## 26. Reconciliation

Even with perfect architecture, distributed systems drift.

Example:

```text
Our Payment DB: PENDING
Provider:        CAPTURED
```

Run periodic reconciliation:

```text
find old PENDING payments
      |
query payment provider
      |
repair state
```

Likewise:

```text
Order CONFIRMED
Inventory reservation still RESERVED
```

A reconciliation job detects and repairs anomalies.

A Staff engineer should assume:

> We will eventually encounter partial failures that weren't anticipated.

Therefore reconciliation is a first-class architecture component, not an afterthought.

---

## 27. Security

For e-commerce:

- TLS everywhere
- Authentication via OAuth/session/JWT
- Authorization for seller/admin operations
- PCI scope minimized
- Don't store raw card details
- Tokenization via payment provider
- Encrypt PII
- Audit critical order/payment state transitions
- Webhook signature verification
- Rate limiting and bot protection
- Fraud/risk engine before payment capture

Payment card details preferably flow directly:

```text
Client
   |
Payment Provider SDK
```

The backend receives a payment token rather than card number.

---

## 28. Observability

Don't just monitor:

```text
CPU
memory
5xx
```

Monitor business invariants.

Examples:

```text
checkout_success_rate
payment_authorization_rate
payment_pending_age
inventory_reservation_failure_rate
order_confirmation_latency
refund_success_rate

orders_paid_but_not_confirmed
orders_confirmed_without_inventory
stale_inventory_reservations
```

Those metrics detect failures that infrastructure metrics won't.

Distributed tracing:

```text
checkout_id
order_id
payment_id
reservation_id
```

should be correlated across services.

---

## 29. API sketch

Core APIs could look like:

```http
GET /products/{productId}

GET /search?q=iphone

POST /cart/items
PUT  /cart/items/{productId}
DELETE /cart/items/{productId}

POST /checkout
Idempotency-Key: abc123

GET /orders/{orderId}

POST /orders/{orderId}/cancel

POST /orders/{orderId}/refund
```

Checkout request:

```json
{
  "cart_id": "C123",
  "shipping_address_id": "A45",
  "payment_method_token": "PM789"
}
```

Response:

```json
{
  "order_id": "O567",
  "status": "PENDING_PAYMENT"
}
```

---

## 30. Critical invariants

At Staff level, explicitly write these on the board.

```text
1. Never sell more inventory than exists.

2. Never charge the customer twice for one payment intent.

3. A successful payment must eventually correspond to an order.

4. A failed/cancelled checkout must eventually release inventory.

5. Order history must remain immutable with respect to historical price.

6. Every state transition must be recoverable/auditable.
```

These invariants are more important than individual technology choices.

---

## 31. Failure scenarios interviewer may ask

### Kafka is unavailable

Checkout shouldn't necessarily fail because analytics/notifications can't consume events.

Outbox retains events.

Publish later.

### Redis is unavailable

Product cache misses hit backing systems.

Cart may temporarily degrade depending on architecture.

Payment/order correctness should never depend exclusively on cache.

### Search is unavailable

Users may be unable to search, but direct product pages and checkout for known items can continue.

### Pricing Service becomes unavailable during checkout

Fail checkout rather than trust a stale cached price beyond an acceptable policy.

Correctness > availability.

### Payment provider becomes slow

Use:

```text
timeouts
circuit breaker
provider routing
async resolution
```

Potentially route:

```text
Provider A unhealthy
      |
Provider B
```

but only when payment state is known.

Never reroute an **UNKNOWN** charge blindly.

---

## 32. One subtle payment problem

Suppose:

```text
Inventory reserved
Payment captured
Order confirmation fails permanently
```

What do we do?

We can't simply lose the payment.

Workflow compensation could:

```text
refund payment
release inventory
mark order failed
```

But often a better business decision is:

```text
recover order creation/confirmation
```

rather than refund automatically.

Staff-level distinction:

> Compensation does not necessarily mean undo every successful step. It means restore a valid business state.

---

## 33. Single diagram to draw in interview

If you had only 5 minutes at the whiteboard, draw this:

```text
                   CDN
                    |
                  BFF/API
                    |
       +------------+-------------+
       |            |             |
    Catalog       Search         Cart
       |            |             |
    Catalog DB  OpenSearch      Redis
       |
       +-------- Events --------+
                    |
                  Kafka
                    |
                Checkout
                    |
          Durable Workflow Engine
                    |
       +------------+-------------+
       |            |             |
    Pricing      Inventory       Order
       |            |             |
      DB            DB            DB
                                   |
                                Outbox
                                   |
                                  Kafka
                                   |
                       +-----------+-----------+
                       |                       |
                    Payment               Fulfillment
                       |
                 Payment Provider
                       |
                    Webhook
```

Then deep dive into **checkout consistency**, because that's where most of the interesting system-design material lies.

---

## 34. Staff vs mid-level answer

A mid-level candidate might say:

> Use microservices, Redis, Kafka, Elasticsearch and SQL.

A Staff-level answer should instead explain **why and where**:

```text
Catalog:
Optimize availability/read throughput.

Search:
Allow eventual consistency.

Cart:
Fast and disposable-ish state.

Order:
Durable transactional state.

Inventory:
Concurrency control around scarce resources.

Payment:
Idempotency + reconciliation.

Cross-service transaction:
Saga/workflow instead of 2PC.

Messaging:
At-least-once + idempotent consumers.

DB/event atomicity:
Transactional outbox.

Multi-region:
Avoid global synchronous coordination where possible.

Operations:
Business-invariant monitoring + reconciliation.
```

That reasoning is much more valuable than naming technologies.

---

## Recommended deep-dive areas

The three areas most worth preparing deeply are:

1. **Checkout consistency**
2. **Inventory concurrency**
3. **Payment failure and reconciliation**

Those three can easily consume 30–40 minutes of a Staff-level interview and expose whether the candidate really understands distributed systems.
