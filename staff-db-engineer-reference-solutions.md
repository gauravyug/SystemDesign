# Reference Design Solutions — Staff Database Engineer 2
### Full solutions for all 5 interview variants — use as your grading reference, not to hand to candidates

Each variant below follows the same structure: Functional Requirements → Non-Functional Requirements → Core Entities → APIs → High-Level Design → Detailed HLD & Deep Dive. Use these to calibrate what a strong staff-level answer looks like; candidates aren't expected to hit every point, but should demonstrate the reasoning patterns.

---

# Variant A — Flash-sale e-commerce
Design the database layer for a high-traffic e-commerce flash-sale platform (e.g. a limited-inventory sneaker drop) that needs to handle a sudden 50x traffic spike

## 1. Functional Requirements
- Browse items with real-time stock count
- Add to cart / attempt purchase during sale window
- Never oversell inventory
- One purchase per (user, item) if per-user limits apply
- Atomic order + payment
- Purchase confirmation/receipt
- Admin: configure sale window, inventory, per-user limits

## 2. Non-Functional Requirements
- 5K TPS steady → 250K TPS peak, ~10 min burst
- p99 < 300ms on buy path
- Strong consistency on inventory decrement
- 99.99% availability during sale
- RPO near-zero, RTO ~2 min
- Fairness / anti-bot protection

## 3. Core Entities
| Entity | Key fields |
|---|---|
| User | id, name, auth_info |
| Item | id, name, price, sale_start, sale_end, total_inventory |
| Inventory | item_id, available_count, version |
| Order | id, user_id, item_id, status, created_at |
| Payment | id, order_id, status, amount, gateway_ref |
| Reservation | id, user_id, item_id, expires_at |

## 4. APIs
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/items/{id}` | Fetch item detail + live stock count |
| POST | `/items/{id}/reserve` | Attempt to reserve 1 unit; returns reservation_id + expiry (idempotency key required) |
| POST | `/orders` | Convert a valid reservation into an order (body: reservation_id, payment_token) |
| GET | `/orders/{id}` | Poll order status |
| POST | `/admin/sales` | Configure a sale window (item_id, start, end, inventory, per_user_limit) |
| GET | `/queue/status` | Poll virtual-queue position (if waiting room is in front of the sale) |

**Design notes**: `reserve` is the only endpoint under real load — it must be idempotent (client-generated idempotency key) since retries under 250K TPS are guaranteed. `/orders` and payment can be slower/async since inventory is already locked by the reservation.

## 5. High-Level Design
```
Client → Edge (CDN, rate limit, virtual queue) → App/order service (stateless)
                                                        │
                                        ┌───────────────┴───────────────┐
                                        ▼                                ▼
                                 Redis cluster                   Postgres primary
                                 (hot inventory counters)        (sharded, source of truth)
                                        │ async reconcile               │
                                        └───────────────►───────────────┘
                                                                         ▼
                                                                 Order queue (Kafka/SQS)
                                                                         ▼
                                                                 Payment service (idempotent)
```

## 6. Detailed HLD & Deep Dive

**Why Redis in front of Postgres**: row-locking 500 hot rows at 250K TPS serializes regardless of shard count, since it's the same rows being hit. Redis absorbs contention via single-threaded atomic `DECR`/Lua check-and-decrement; Postgres remains the durable ledger, updated async.

**Reservation flow**: `reserve` does a Lua script in Redis (atomic check `available > 0` then decrement + set TTL key for the hold). On success, emits an event to the order queue for async Postgres write. On reservation expiry (TTL), a background job increments the Redis counter back.

**Sharding**: hash-partition inventory/orders by `item_id` so the 500 hot items spread across shards rather than clustering on one. A single mega-hot item (e.g. the single most-wanted SKU) may still need sub-sharding of its counter (e.g. N Redis keys with a fan-in reconciler) if it alone exceeds one shard's throughput.

**Consistency edge case**: Redis says available, Postgres write fails — reconciliation job must detect the mismatch (compare Redis counter to Postgres row periodically) and either retry the write or roll back the Redis decrement. This must be idempotent and monitored — silent oversell/underselling here is the single most damaging failure mode.

**DR**: WAL-based physical replication for near-zero RPO. Redis needs its own replica (Redis Sentinel/Cluster) — if Redis is lost mid-sale with no Postgres write yet, in-flight reservations are lost; mitigate with a short async write-ahead log to Postgres per reservation attempt, not just on success.

**Indexing/proxy**: PgBouncer transaction pooling given connection volume; partial index on `items.sale_active = true` for browse queries; separate read replicas for the (much higher volume, less contended) browse/search path.

**PostgreSQL specifics to probe**: autovacuum tuning under bursty writes, `SERIALIZABLE` vs advisory locks for the fallback path if Redis is bypassed, WAL archiving cadence vs RPO target.

---

# Variant B — Concert/movie ticket booking
Design the database layer for a ticket booking platform where tickets for a single stadium concert (60,000 seats) go on sale simultaneously to 3 million waiting users

## 1. Functional Requirements
- Display seat map with real-time availability
- Hold a seat for checkout (5 min TTL)
- Confirm purchase; hold converts to sold or expires back to available
- No seat sold twice
- Refund/cancellation flow releases seat back to inventory (post-sale)

## 2. Non-Functional Requirements
- 3M concurrent users at sale-open, 60K seats total
- p99 < 300ms on seat-hold path
- Strong consistency per-seat (not per-item-counter — this is the key difference from Variant A)
- RPO near-zero, RTO ~2 min
- No seat double-booked even under retries/duplicate requests

## 3. Core Entities
| Entity | Key fields |
|---|---|
| Event | id, venue_id, start_time, total_seats |
| Seat | id, event_id, seat_label, status (AVAILABLE/HELD/SOLD) |
| Hold | id, seat_id, user_id, expires_at |
| Order | id, user_id, seat_ids[], status |
| Payment | id, order_id, status, amount |

## 4. APIs
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/events/{id}/seats` | Live seat map (cacheable, short TTL) |
| POST | `/seats/{id}/hold` | Atomically hold a seat if AVAILABLE (idempotency key) |
| POST | `/orders` | Convert a set of holds into an order + trigger payment |
| DELETE | `/seats/{id}/hold` | Explicit release (user backs out) |
| GET | `/orders/{id}` | Poll status |

**Design notes**: unlike Variant A's counter, this is **per-row** contention across 60K distinct seats rather than 500 shared counters — spreads load naturally, but the seat map read (`GET /seats`) is a massive fan-out read problem at 3M concurrent users, which is its own distinct scaling axis.

## 5. High-Level Design
```
Client → Edge (rate limit, virtual queue) → App/booking service
                                                    │
                                    ┌───────────────┴───────────────┐
                                    ▼                                ▼
                             Redis (seat-state cache,          Postgres primary
                             per-seat keys + pub/sub            (seats table,
                             for live map updates)              sharded by event_id)
                                    │                                │
                                    └── periodic reconcile ──────────┘
                                                                      ▼
                                                              Order/Payment service
```

## 6. Detailed HLD & Deep Dive

**Seat hold = per-row optimistic lock**: `UPDATE seats SET status='HELD', held_by=?, expires_at=? WHERE id=? AND status='AVAILABLE'` — a single-row conditional update is naturally safe under Postgres MVCC without needing Redis for correctness, unlike Variant A's shared counter. Redis here is primarily a **read cache + pub/sub layer** for the seat map, not the source of truth for correctness.

**Why this differs architecturally from Variant A**: because contention is spread across 60K rows instead of 500, Postgres alone (with good indexing) may handle the write path without a Redis-decrement pattern — a candidate who defaults to "put everything in Redis" without noticing this distinction is pattern-matching, not reasoning. The interesting scaling problem here is the **read fan-out** (3M clients polling/subscribing to a live seat map), which is a caching/pub-sub problem, not a write-contention problem.

**Expiry of holds**: background sweeper (`expires_at < now()`) resets HELD → AVAILABLE; at scale this needs to be a partitioned/batched job, not a naive full-table scan — index on `(status, expires_at)`.

**Sharding**: by `event_id` — each concert is independent, so this shards cleanly with no cross-shard transactions needed (a real advantage over Variant A).

**DR**: since correctness lives in Postgres row state directly (not an external cache), DR is more standard — WAL shipping, replica promotion. Redis loss here is not catastrophic (rebuild seat-map cache from Postgres), a meaningfully different risk profile than Variant A.

**PostgreSQL specifics to probe**: `SELECT ... FOR UPDATE SKIP LOCKED` for hold-assignment patterns, index design on `(event_id, status)`, connection pooling for the 3M-concurrent read fan-out (likely needs a caching layer/CDN in front of Postgres reads entirely, not just a pooler).

---

# Variant C — Ride-hailing surge/driver matching
Design the database layer for a ride-hailing platform during a surge event (e.g. New Year's Eve) where driver supply in a dense city area is far outstripped by rider demand.

## 1. Functional Requirements
- Rider requests a ride; system matches to nearest available driver
- A driver can be assigned to only one active ride at a time
- Ride state transitions: REQUESTED → MATCHED → IN_PROGRESS → COMPLETED
- Surge pricing calculated per geo-zone in real time
- Cancellation releases driver back to available pool

## 2. Non-Functional Requirements
- 500K requests/min in one dense city zone vs ~20K available drivers
- Strong consistency on driver-assignment (no double-assignment)
- Multi-region: drivers/riders geo-partitioned, platform globally distributed
- Low-latency matching (p99 < 500ms for match decision)
- RTO/RPO similar to other variants; ride-state loss is a real financial/dispute risk

## 3. Core Entities
| Entity | Key fields |
|---|---|
| Driver | id, current_location, status (AVAILABLE/ON_TRIP/OFFLINE), zone_id |
| Rider | id, current_location |
| Ride | id, rider_id, driver_id, status, requested_at, zone_id |
| Zone | id, geo_boundary, current_surge_multiplier |

## 4. APIs
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/rides/request` | Rider requests a ride (lat/lng); triggers matching |
| POST | `/drivers/{id}/location` | High-frequency driver location ping (write-heavy, separate path) |
| POST | `/rides/{id}/accept` | Driver accepts a match candidate |
| PATCH | `/rides/{id}/status` | State transitions (start trip, complete, cancel) |
| GET | `/zones/{id}/surge` | Current surge multiplier for a zone |

**Design notes**: `/drivers/{id}/location` is extremely high write volume (every few seconds per driver) but low-consistency-requirement (eventual consistency is fine) — a deliberately different profile from `/rides/{id}/accept`, which needs strong consistency. A strong candidate should design **different consistency tiers for different endpoints** rather than one blanket strategy.

## 5. High-Level Design
```
Rider/Driver apps → Edge (regional routing by geo)
                          │
              ┌───────────┴────────────┐
              ▼                         ▼
    Location service (Redis geo-index,  Matching service (stateless)
    eventual consistency, high write)          │
              │                                 ▼
              └──── nearest-driver query ──► Postgres primary
                                             (rides table, sharded by zone_id)
                                                       │
                                                       ▼
                                             Async: surge pricing engine,
                                             notification service
```

## 6. Detailed HLD & Deep Dive

**Two distinct data paths**: driver location (high-write, eventually-consistent, lives in Redis geo-index / geohash structure — `GEOADD`/`GEORADIUS`) vs. ride/driver-assignment state (low-write-per-event but strongly-consistent, lives in Postgres). Conflating these into one consistency model is the most common design mistake here — good candidates should explicitly separate them.

**Driver assignment race**: multiple ride requests may target the same nearby driver simultaneously — assignment must be atomic: `UPDATE drivers SET status='ON_TRIP' WHERE id=? AND status='AVAILABLE'`, same conditional-update pattern as Variant B's seat hold. First writer wins; losers re-query for next-nearest driver.

**Sharding**: by `zone_id`/geography — naturally aligns with how the business operates (a driver in Bengaluru never needs cross-shard consistency with a ride in Mumbai). Cross-zone edge cases (driver near a zone boundary) are the interesting deep-dive question.

**Geo-index specifics**: Redis geospatial commands, or a dedicated geo-index (H3/S2 cell-based sharding) if scale demands more than Redis's built-in geo support — good candidates may bring this up.

**DR**: region-scoped — since zones are geographically partitioned, DR strategy should be per-region rather than one global failover, which changes the RTO/RPO discussion meaningfully from a single-region design.

**PostgreSQL specifics to probe**: PostGIS extension for geo-queries if geo logic is pushed into Postgres rather than Redis; partial indexes on `status='AVAILABLE'`; `SERIALIZABLE` vs conditional-update tradeoffs for the assignment race.

---

# Variant D — Flash discount / food delivery surge
Design the database layer for a food delivery app running a 1-hour flash discount (70% off) that causes a 40x spike in order volume across a small set of highly popular restaurants

## 1. Functional Requirements
- Display discounted restaurants + real-time remaining order-capacity
- Place order against a restaurant's capacity cap
- Never exceed a restaurant's hourly fulfillment cap
- Atomic order + payment + capacity decrement
- Restaurant can adjust capacity mid-event (rare, but must be handled safely)

## 2. Non-Functional Requirements
- 8K TPS steady → 320K TPS peak, sustained for a full 60 minutes (longer than A/B/C)
- Strong consistency on capacity decrement
- RPO near-zero, RTO ~2 min
- Must distinguish short-burst mitigation from genuinely sustained load handling

## 3. Core Entities
| Entity | Key fields |
|---|---|
| Restaurant | id, hourly_capacity, current_orders_this_hour |
| MenuItem | id, restaurant_id, price, discount_pct |
| Order | id, user_id, restaurant_id, items[], status |
| Payment | id, order_id, status, amount |
| CapacityWindow | restaurant_id, window_start, orders_count, cap |

## 4. APIs
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/restaurants/{id}` | Restaurant detail + live remaining capacity |
| POST | `/orders` | Place order (idempotency key), decrements capacity atomically |
| GET | `/orders/{id}` | Poll status |
| PATCH | `/admin/restaurants/{id}/capacity` | Adjust hourly cap mid-event |

## 5. High-Level Design
```
Client → Edge (rate limit, queue) → App/order service
                                          │
                          ┌───────────────┴───────────────┐
                          ▼                                ▼
                   Redis cluster                    Postgres primary
                   (capacity counters,               (sharded by
                   per restaurant, TTL'd              restaurant_id)
                   per hour window)
                          │ async reconcile                │
                          └────────────────►────────────────┘
                                                             ▼
                                                     Order queue → Payment service
```

## 6. Detailed HLD & Deep Dive

**Structurally similar to Variant A** (Redis-hot-counter + Postgres-durable pattern) — the deliberate difference is **duration**: a 60-minute sustained 320K TPS load can't be treated purely as "absorb the burst in memory and catch up later" the way a 10-minute spike can. A strong candidate should flag that sustained load needs genuine horizontal write scaling (more shards, more Redis nodes) rather than relying solely on buffering — buffering delays the problem, it doesn't solve throughput for an hour straight.

**Capacity window semantics**: capacity resets hourly — modeling this as a TTL'd Redis key (`restaurant:{id}:capacity:{hour}`) with a fresh window each hour is cleaner than a single mutable counter with manual resets, and avoids a "midnight rollover" class of bug.

**Mid-event capacity adjustment**: raising a restaurant's cap mid-hour is a straightforward Redis `INCRBY`; lowering it is trickier — if `current_orders > new_cap`, decide whether to honor already-placed orders (recommended) and just stop accepting new ones, rather than trying to retroactively cancel.

**Sharding**: by `restaurant_id`, same rationale as item-level sharding in Variant A.

**DR**: same WAL/replication approach as A, but the longer sustained window means a mid-event failover is more likely to actually occur during the event (vs. a 10-min burst where failover risk is lower simply due to shorter exposure window) — worth having the candidate reason about this probability difference.

**PostgreSQL specifics to probe**: same as Variant A, plus discussion of whether `current_orders_this_hour` should be a materialized counter vs. computed via `COUNT()` (should be a counter — `COUNT()` under this load is a non-starter, good candidates should say so immediately).

---

# Variant E — Banking/payments ledger burst (hardest variant)
Design the database layer for a digital bank's fund-transfer system during a government subsidy disbursement event, where 10 million accounts receive a credit simultaneously and a large fraction of recipients attempt to withdraw/transfer within the same hour."

## 1. Functional Requirements
- Credit 10M accounts simultaneously (batch disbursement)
- Support withdrawal/transfer immediately after credit
- Full auditability — every transaction traceable, no silent loss
- Double-spend structurally impossible

## 2. Non-Functional Requirements
- 10M credit transactions must land atomically
- Peak withdrawal/transfer ~180K TPS for ~45 min
- **RPO = zero** (not near-zero) — hardest constraint of all 5 variants
- Regulatory: no eventual consistency permitted on the ledger

## 3. Core Entities
| Entity | Key fields |
|---|---|
| Account | id, balance, version |
| LedgerEntry | id, account_id, amount, type (CREDIT/DEBIT), reference_id, created_at |
| Transfer | id, from_account, to_account, amount, status, idempotency_key |
| DisbursementBatch | id, total_accounts, status, created_at |

## 4. APIs
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/admin/disbursements` | Trigger a batch credit job |
| POST | `/transfers` | Initiate a transfer/withdrawal (idempotency key mandatory) |
| GET | `/accounts/{id}/balance` | Current balance (must reflect committed ledger state only) |
| GET | `/accounts/{id}/ledger` | Full auditable transaction history |

## 5. High-Level Design
```
Batch disbursement job → Postgres primary (ledger table, sharded by account_id)
                                    │
Client → Edge (rate limit) → App/transfer service ──► Postgres primary
                                                            │
                                              Synchronous replica (for RPO=0)
                                                            │
                                                     Audit/reporting replica (async, read-only)
```

**No Redis hot-counter layer** — this is the intended twist. A candidate reaching for the Variant-A pattern here should be pushed on it directly: *"You said RPO must be zero. If Redis holds the authoritative balance and Redis dies before reconciling to Postgres, what happened to that money?"*

## 6. Detailed HLD & Deep Dive

**Why this variant rules out the Redis-hot-counter default**: every other variant tolerates a small window of "eventually consistent" state (a slightly-stale inventory count is a UX problem; a slightly-stale bank balance is a compliance and possibly criminal-liability problem). RPO=0 means every acknowledged write must be durably committed — synchronously replicated — before responding success. This is the whole point of the variant: testing whether a candidate can recognize when a familiar pattern doesn't apply.

**Double-entry ledger, not mutable balance**: rather than `UPDATE accounts SET balance = balance - X`, model every movement as an immutable `LedgerEntry` row (debit + credit pair), and compute balance as a derived sum (with a periodically-materialized snapshot for read performance). This gives natural auditability and makes double-spend detection a matter of checking for duplicate `idempotency_key`/`reference_id`, not race-prone balance arithmetic.

**Batch credit at 10M accounts**: not a single transaction — batched into chunks (e.g. 10K accounts per transaction) to avoid one gigantic lock-holding transaction; must be idempotent/resumable if the job crashes partway (track progress in `DisbursementBatch`, replay from last committed chunk).

**Concurrency control for transfers**: `SERIALIZABLE` isolation or explicit row versioning (`SELECT ... FOR UPDATE` on both accounts in a transfer, always locked in a consistent order — e.g. by ascending account_id — to avoid deadlocks) is the correct tool here, not optimistic Redis counters.

**Sharding**: by `account_id`, but transfers *between* accounts on different shards break the simple single-shard-transaction model — this is the deepest technical question in the whole kit. Options to probe: two-phase commit (slow, complex), a saga pattern with compensating transactions (eventual consistency on the cross-shard case specifically, which needs to be justified against the "no eventual consistency" requirement — a good candidate will flag this tension explicitly), or keeping frequently-paired accounts co-located on the same shard.

**Sync replication for RPO=0**: at least one synchronous replica confirms write durability before ack — this trades some write latency for the hard durability guarantee; candidate should explicitly acknowledge this p99 latency cost versus the other variants' async-replication approach.

**PostgreSQL specifics to probe**: `SERIALIZABLE` isolation deeply (this is the variant where it actually matters, not just as a mentioned term), synchronous_commit settings, two-phase commit (`PREPARE TRANSACTION`) if cross-shard transfers come up, WAL sync guarantees, deadlock detection/ordering strategy for multi-account locks.

---

## Cross-Variant Summary Table (quick reference while grading)

| Variant | Contention shape | Consistency escape hatch? | Sharding key | Standout deep-dive question |
|---|---|---|---|---|
| A — Flash sale | Few hot rows (~500) | Yes — Redis + async reconcile | item_id | What happens when Redis and Postgres disagree? |
| B — Ticket booking | Many rows (60K), read fan-out | Partial — Postgres alone can work | event_id | Is Redis even necessary for correctness here? |
| C — Ride-hailing | Two consistency tiers (location vs assignment) | Yes for location, no for assignment | zone_id/geo | Why do location writes and ride-state writes need different models? |
| D — Food delivery | Few hot rows, sustained 60 min | Yes, but buffering alone isn't enough | restaurant_id | Burst-absorb vs. genuine horizontal scaling |
| E — Banking | Cross-shard transfers, RPO=0 | **No** — this is the point | account_id | How do you handle a transfer across two shards without eventual consistency? |
