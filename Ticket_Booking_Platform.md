# Staff Database Engineer — System Design Round
## Ticket Booking Platform (PostgreSQL-focused)

**Mandate (from the interview rubric — strictly enforced in this doc):**
1. PostgreSQL must be covered extensively — not just named. Every architectural claim below is tied to a specific Postgres mechanism (MVCC, autovacuum, `synchronous_standby_names`, isolation levels, PgBouncer pooling, Patroni failover, WAL-based PITR).
2. This is a **staff-level Database Engineer** round, not a generalist system-design round — the bar is depth on database internals and tradeoffs, not breadth of the whole stack.
3. Every rubric pillar (Data-Driven Design, Scalable DB Architecture, Resiliency & DR, Building for Scale, Communication) must be addressed — the scoring sheet in Part 3 mirrors them 1:1.
4. Feedback must call out mandate compliance explicitly — see Part 3.

---

## PART 1 — Problem Statement to Give the Candidate

> **Prompt (read aloud / share verbatim):**
>
> "Design the database layer for a ticket booking platform. A single stadium concert with 60,000 seats goes on sale, and 3 million users are waiting to buy tickets the moment sales open.
>
> Walk me through how you'd architect the data layer to handle this — I want to hear how you'd characterize the workload, how you'd design the PostgreSQL schema and architecture to survive the contention, how you'd think about resiliency and failure recovery, and how the whole thing scales under this specific load pattern.
>
> Assume PostgreSQL is the primary datastore. You can use supporting technologies (caching, queues, etc.) but I want the core data architecture — schema, consistency model, replication, and scaling strategy — to be justified from first principles, not just named."

**Optional follow-up prompts to probe deeper (use if candidate stalls or to test depth):**
1. "What's the actual bottleneck here — is it storage, throughput, or something else? Convince me."
2. "Would you shard this data? Why or why not?"
3. "Walk me through exactly what happens in Postgres when 5,000 users try to buy the same seat in the same second."
4. "Your primary database node dies 30 seconds into the sale. What happens to in-flight orders? What's your RPO/RTO here?"
5. "How would autovacuum/MVCC bloat show up in this workload, and what would you do about it?"
6. "At what point, if any, would you actually need to shard or partition — describe the trigger condition, not just the technique."

**What a strong candidate does in the first 60 seconds:** names the real bottleneck (3M actors vs. 60K rows = contention, not data volume) *before* touching schema or API design. This is the single strongest signal of seniority — weak candidates jump straight to `CREATE TABLE` and REST endpoints.

---

## PART 1.5 — High-Level Design

**Request flow, top to bottom:**

```
[Clients — 3M concurrent users]
            │
[Waiting room — token-based admission queue]      ← rate-limits entry, not DB-backed
            │
[API gateway — rate limiting & routing]
            │
[Redis — seat hold arbitration (CAS)]              ← absorbs ~99% of write attempts
            │  (only arbitration winners proceed)
[PgBouncer — transaction pooling]
            │
[Postgres primary — CAS update + orders insert]
            │
      ┌─────┴─────┐
      ▼           ▼
[Sync replica]  [Async replicas]
 orders durability   seat-map reads
 RPO ≈ 0             (staleness tolerated)
```

**What each hop is responsible for, and why it's there:**

| Component | Responsibility | Why it exists in this design specifically |
|---|---|---|
| Waiting room | Admits users at a rate the backend can sustain via queue tokens | Turns an uncontrolled 3M-request spike into a controlled, rate-limited stream *before* anything is DB-adjacent |
| API gateway | Routing, rate limiting, request validation | Standard edge layer; also where the waiting-room token is checked |
| Redis | Atomic CAS hold per seat, TTL-based | This is the real arbitration point — resolves the seat-level race in-memory, so Postgres never sees the losing 99% of attempts |
| PgBouncer | Transaction-mode connection pooling | Postgres's per-connection process model can't absorb thousands of simultaneous backend connections; PgBouncer multiplexes them |
| Postgres primary | Source of truth for `orders`, `event_seat_inventory`, final CAS + commit | Only ever receives writes from users who already won the Redis hold |
| Sync replica | Durability for `orders`/payment rows | RPO ≈ 0 requirement — can't lose a committed order on failover |
| Async replicas | Serve seat-map read traffic | Reads tolerate ~1s staleness, so they're isolated from the write-critical path entirely |

**Deliberately left out of the diagram, but worth stating explicitly when asked:**
- A **reconciliation worker** runs asynchronously between Redis and Postgres to heal orphaned holds (e.g. a crash between winning a Redis hold and writing the pending order row). Left off the main diagram to keep the hot path readable — call it out proactively in the resiliency discussion.
- A **payment provider** sits between the hold and the final `held → sold` transition, external to the data layer.

---

## PART 2 — Model Solution, Mapped to Rubric

### Pillar 1 — Data-Driven System Design (25%)
*Workload characterization, TPS, latency, access patterns, capacity planning*

| Dimension | Value | Design implication |
|---|---|---|
| Concurrent actors | 3M | Drives admission control, not schema |
| Contended keyspace | 60K rows | Confirms this is a **contention** problem, not a storage-scale problem |
| Peak QPS on hottest single row | 1,000s–10,000s of CAS attempts/sec (front-row seat) | Single-row lock contention is the real worst case, not aggregate throughput |
| Read:write ratio | ~100:1 (seat-map browsing vs. actual purchase attempts) | Reads can be served from replicas/cache; writes need careful arbitration |
| Consistency need | Writes: strict, no double-sell. Reads: tolerate ~1s staleness | Justifies splitting consistency models by path (sync writes, async reads) |
| Burst shape | Seconds-to-minutes extreme spike, then falls to near-zero | Provision for the spike explicitly; don't design for steady-state average |
| RPO/RTO | Near-zero RPO on `orders`/payments (money); seat-hold state can tolerate seconds of loss | Drives replication topology in Pillar 3 |

**What to say out loud:** *"60K rows fits in cache. The problem isn't data volume, it's that up to 3 million writers are racing to update a keyspace that small in a period of seconds — so my entire design is organized around shedding and arbitrating contention before it reaches Postgres's transactional core."*

---

### Pillar 2 — Scalable Database Architecture (25%)
*Replication, partitioning, sharding, read/write scaling, consistency models, HA — Postgres-specific*

**Schema:**
```sql
CREATE TABLE event_seat_inventory (
    event_id    BIGINT,
    seat_id     BIGINT,
    status      SMALLINT NOT NULL DEFAULT 0,   -- 0 avail, 1 held, 2 sold
    held_by     BIGINT,
    held_until  TIMESTAMPTZ,
    version     INT NOT NULL DEFAULT 0,
    PRIMARY KEY (event_id, seat_id)
) PARTITION BY HASH (event_id);

CREATE TABLE orders (
    order_id     BIGINT PRIMARY KEY,
    user_id      BIGINT,
    event_id     BIGINT,
    status       SMALLINT,
    created_at   TIMESTAMPTZ DEFAULT now(),
    total_amount NUMERIC(10,2)
);
```

**Atomic hold (single round trip, no separate lock+update):**
```sql
UPDATE event_seat_inventory
SET status = 1, held_by = $1, held_until = now() + interval '5 minutes', version = version + 1
WHERE event_id = $2 AND seat_id = $3
  AND status = 0 AND version = $4;
-- rows affected = 0 is the expected, common case — not an error
```

**Postgres-specific points to hit:**
- **MVCC/bloat risk**: failed CAS attempts still generate dead tuples under MVCC. At 3M attempts vs. 60K rows this bloats fast — call this out explicitly and either tune `autovacuum` aggressively on this table or (better) keep failed attempts out of Postgres entirely by arbitrating holds in Redis first, writing to Postgres only for winners.
- **Connection scaling**: Postgres's per-connection process model can't absorb 3M connections. **PgBouncer in transaction-pooling mode** sits in front, backed by a much smaller real backend pool — name this explicitly, it's the standard answer.
- **Partitioning**: hash-partition `event_seat_inventory` by `event_id` — not because 60K rows for one event needs it, but for isolating hot partitions and scoping autovacuum/index maintenance across many concurrent events in production.
- **Replication & consistency split**: `synchronous_standby_names` (quorum sync) for the `orders`/payment path — can't lose a committed order. Async streaming replicas fanned out for seat-map read traffic, which tolerates staleness. This is the concrete answer to "consistency models": **strong on writes, relaxed on reads, by design, not by accident.**
- **Isolation level**: `READ COMMITTED` with the CAS `WHERE` clause is sufficient — `SERIALIZABLE` would add unnecessary abort/retry overhead here since correctness is already enforced at the row level. Explaining why you're *not* reaching for the strongest isolation level is a strong signal.

---

### Pillar 3 — System Resiliency & Disaster Recovery (25%)
*BCP, failover, backup/recovery, replica sync, RTO/RPO*

- **RPO/RTO by table, not globally**: `orders`/`payments` → RPO ≈ 0 via synchronous replication; `event_seat_inventory` hold-state → RPO of a few seconds is acceptable since Redis is the real-time source of truth and Postgres is reconciled async.
- **Failover mechanism**: Patroni (leader election) backed by etcd/Consul for consensus — name it specifically, it's the standard Postgres HA pattern and shows operational exposure beyond textbook HA theory.
- **Backups**: continuous WAL archiving via `pgBackRest` or `WAL-G`, periodic base backups, enabling point-in-time recovery — necessary because a bad deploy or corruption mid-sale needs PITR, not a nightly snapshot.
- **Split-brain avoidance**: fencing via Patroni/etcd leases, not just "we have replicas."
- **Failure scenario tied to this workload**: if the primary dies mid-burst, quorum-sync replicas mean no committed order is lost on promotion — but in-flight Redis holds referencing the old generation need an explicit reconciliation pass after failover. Naming this specific reconciliation step (not just "we failover") is what separates a strong answer from a generic one.

---

### Pillar 4 — Building for Scale (25%)
# Staff Database Engineer — System Design Round
## Ticket Booking Platform (PostgreSQL-focused)

**Mandate (from the interview rubric — strictly enforced in this doc):**
1. PostgreSQL must be covered extensively — not just named. Every architectural claim below is tied to a specific Postgres mechanism (MVCC, autovacuum, `synchronous_standby_names`, isolation levels, PgBouncer pooling, Patroni failover, WAL-based PITR).
2. This is a **staff-level Database Engineer** round, not a generalist system-design round — the bar is depth on database internals and tradeoffs, not breadth of the whole stack.
3. Every rubric pillar (Data-Driven Design, Scalable DB Architecture, Resiliency & DR, Building for Scale, Communication) must be addressed — the scoring sheet in Part 3 mirrors them 1:1.
4. Feedback must call out mandate compliance explicitly — see Part 3.

---

## PART 1 — Problem Statement to Give the Candidate

> **Prompt (read aloud / share verbatim):**
>
> "Design the database layer for a ticket booking platform. A single stadium concert with 60,000 seats goes on sale, and 3 million users are waiting to buy tickets the moment sales open.
>
> Walk me through how you'd architect the data layer to handle this — I want to hear how you'd characterize the workload, how you'd design the PostgreSQL schema and architecture to survive the contention, how you'd think about resiliency and failure recovery, and how the whole thing scales under this specific load pattern.
>
> Assume PostgreSQL is the primary datastore. You can use supporting technologies (caching, queues, etc.) but I want the core data architecture — schema, consistency model, replication, and scaling strategy — to be justified from first principles, not just named."

**Optional follow-up prompts to probe deeper (use if candidate stalls or to test depth):**
1. "What's the actual bottleneck here — is it storage, throughput, or something else? Convince me."
2. "Would you shard this data? Why or why not?"
3. "Walk me through exactly what happens in Postgres when 5,000 users try to buy the same seat in the same second."
4. "Your primary database node dies 30 seconds into the sale. What happens to in-flight orders? What's your RPO/RTO here?"
5. "How would autovacuum/MVCC bloat show up in this workload, and what would you do about it?"
6. "At what point, if any, would you actually need to shard or partition — describe the trigger condition, not just the technique."

**What a strong candidate does in the first 60 seconds:** names the real bottleneck (3M actors vs. 60K rows = contention, not data volume) *before* touching schema or API design. This is the single strongest signal of seniority — weak candidates jump straight to `CREATE TABLE` and REST endpoints.

---

## PART 1.5 — High-Level Design
<img width="818" height="638" alt="image" src="https://github.com/user-attachments/assets/6124a0ac-a4dc-4ab6-bccf-ff3e9d21eeb1" />

**Request flow, top to bottom:**

```
[Clients — 3M concurrent users]
            │
[Waiting room — token-based admission queue]      ← rate-limits entry, not DB-backed
            │
[API gateway — rate limiting & routing]
            │
[Redis — seat hold arbitration (CAS)]              ← absorbs ~99% of write attempts
            │  (only arbitration winners proceed)
[PgBouncer — transaction pooling]
            │
[Postgres primary — CAS update + orders insert]
            │
      ┌─────┴─────┐
      ▼           ▼
[Sync replica]  [Async replicas]
 orders durability   seat-map reads
 RPO ≈ 0             (staleness tolerated)
```

**What each hop is responsible for, and why it's there:**

| Component | Responsibility | Why it exists in this design specifically |
|---|---|---|
| Waiting room | Admits users at a rate the backend can sustain via queue tokens | Turns an uncontrolled 3M-request spike into a controlled, rate-limited stream *before* anything is DB-adjacent |
| API gateway | Routing, rate limiting, request validation | Standard edge layer; also where the waiting-room token is checked |
| Redis | Atomic CAS hold per seat, TTL-based | This is the real arbitration point — resolves the seat-level race in-memory, so Postgres never sees the losing 99% of attempts |
| PgBouncer | Transaction-mode connection pooling | Postgres's per-connection process model can't absorb thousands of simultaneous backend connections; PgBouncer multiplexes them |
| Postgres primary | Source of truth for `orders`, `event_seat_inventory`, final CAS + commit | Only ever receives writes from users who already won the Redis hold |
| Sync replica | Durability for `orders`/payment rows | RPO ≈ 0 requirement — can't lose a committed order on failover |
| Async replicas | Serve seat-map read traffic | Reads tolerate ~1s staleness, so they're isolated from the write-critical path entirely |

**Deliberately left out of the diagram, but worth stating explicitly when asked:**
- A **reconciliation worker** runs asynchronously between Redis and Postgres to heal orphaned holds (e.g. a crash between winning a Redis hold and writing the pending order row). Left off the main diagram to keep the hot path readable — call it out proactively in the resiliency discussion.
- A **payment provider** sits between the hold and the final `held → sold` transition, external to the data layer.

---

## PART 2 — Model Solution, Mapped to Rubric

### Pillar 1 — Data-Driven System Design (25%)
*Workload characterization, TPS, latency, access patterns, capacity planning*

| Dimension | Value | Design implication |
|---|---|---|
| Concurrent actors | 3M | Drives admission control, not schema |
| Contended keyspace | 60K rows | Confirms this is a **contention** problem, not a storage-scale problem |
| Peak QPS on hottest single row | 1,000s–10,000s of CAS attempts/sec (front-row seat) | Single-row lock contention is the real worst case, not aggregate throughput |
| Read:write ratio | ~100:1 (seat-map browsing vs. actual purchase attempts) | Reads can be served from replicas/cache; writes need careful arbitration |
| Consistency need | Writes: strict, no double-sell. Reads: tolerate ~1s staleness | Justifies splitting consistency models by path (sync writes, async reads) |
| Burst shape | Seconds-to-minutes extreme spike, then falls to near-zero | Provision for the spike explicitly; don't design for steady-state average |
| RPO/RTO | Near-zero RPO on `orders`/payments (money); seat-hold state can tolerate seconds of loss | Drives replication topology in Pillar 3 |

**What to say out loud:** *"60K rows fits in cache. The problem isn't data volume, it's that up to 3 million writers are racing to update a keyspace that small in a period of seconds — so my entire design is organized around shedding and arbitrating contention before it reaches Postgres's transactional core."*

---

### Pillar 2 — Scalable Database Architecture (25%)
*Replication, partitioning, sharding, read/write scaling, consistency models, HA — Postgres-specific*

**Schema:**
```sql
CREATE TABLE event_seat_inventory (
    event_id    BIGINT,
    seat_id     BIGINT,
    status      SMALLINT NOT NULL DEFAULT 0,   -- 0 avail, 1 held, 2 sold
    held_by     BIGINT,
    held_until  TIMESTAMPTZ,
    version     INT NOT NULL DEFAULT 0,
    PRIMARY KEY (event_id, seat_id)
) PARTITION BY HASH (event_id);

CREATE TABLE orders (
    order_id     BIGINT PRIMARY KEY,
    user_id      BIGINT,
    event_id     BIGINT,
    status       SMALLINT,
    created_at   TIMESTAMPTZ DEFAULT now(),
    total_amount NUMERIC(10,2)
);
```

**Atomic hold (single round trip, no separate lock+update):**
```sql
UPDATE event_seat_inventory
SET status = 1, held_by = $1, held_until = now() + interval '5 minutes', version = version + 1
WHERE event_id = $2 AND seat_id = $3
  AND status = 0 AND version = $4;
-- rows affected = 0 is the expected, common case — not an error
```

**Postgres-specific points to hit:**
- **MVCC/bloat risk**: failed CAS attempts still generate dead tuples under MVCC. At 3M attempts vs. 60K rows this bloats fast — call this out explicitly and either tune `autovacuum` aggressively on this table or (better) keep failed attempts out of Postgres entirely by arbitrating holds in Redis first, writing to Postgres only for winners.
- **Connection scaling**: Postgres's per-connection process model can't absorb 3M connections. **PgBouncer in transaction-pooling mode** sits in front, backed by a much smaller real backend pool — name this explicitly, it's the standard answer.
- **Partitioning**: hash-partition `event_seat_inventory` by `event_id` — not because 60K rows for one event needs it, but for isolating hot partitions and scoping autovacuum/index maintenance across many concurrent events in production.
- **Replication & consistency split**: `synchronous_standby_names` (quorum sync) for the `orders`/payment path — can't lose a committed order. Async streaming replicas fanned out for seat-map read traffic, which tolerates staleness. This is the concrete answer to "consistency models": **strong on writes, relaxed on reads, by design, not by accident.**
- **Isolation level**: `READ COMMITTED` with the CAS `WHERE` clause is sufficient — `SERIALIZABLE` would add unnecessary abort/retry overhead here since correctness is already enforced at the row level. Explaining why you're *not* reaching for the strongest isolation level is a strong signal.

---

### Pillar 3 — System Resiliency & Disaster Recovery (25%)
*BCP, failover, backup/recovery, replica sync, RTO/RPO*

- **RPO/RTO by table, not globally**: `orders`/`payments` → RPO ≈ 0 via synchronous replication; `event_seat_inventory` hold-state → RPO of a few seconds is acceptable since Redis is the real-time source of truth and Postgres is reconciled async.
- **Failover mechanism**: Patroni (leader election) backed by etcd/Consul for consensus — name it specifically, it's the standard Postgres HA pattern and shows operational exposure beyond textbook HA theory.
- **Backups**: continuous WAL archiving via `pgBackRest` or `WAL-G`, periodic base backups, enabling point-in-time recovery — necessary because a bad deploy or corruption mid-sale needs PITR, not a nightly snapshot.
- **Split-brain avoidance**: fencing via Patroni/etcd leases, not just "we have replicas."
- **Failure scenario tied to this workload**: if the primary dies mid-burst, quorum-sync replicas mean no committed order is lost on promotion — but in-flight Redis holds referencing the old generation need an explicit reconciliation pass after failover. Naming this specific reconciliation step (not just "we failover") is what separates a strong answer from a generic one.

---

### Pillar 4 — Building for Scale (25%)

The funnel — explicitly framed as "keeping load off Postgres":

```
[Waiting room / token admission]   ← rate-limits entry, not DB-backed
        ↓
[Redis: seat availability + CAS hold]   ← absorbs ~99% of write attempts
        ↓ (only winners proceed)
[PgBouncer — transaction pooling]
        ↓
[Postgres primary — single-statement CAS UPDATE → orders insert]
        ↓ async replication
[Sync replica(s): durability for orders] + [Async replicas: seat-map reads]
```

**Closing line to say out loud (ties all four pillars together):**
*"The core scaling move is that Postgres only ever sees writes from users who already won a Redis-arbitrated hold — so a 3-million-way write storm at the edge becomes a few-thousand-QPS write load by the time it reaches the database. That's a number a well-tuned Postgres primary, with PgBouncer and short single-statement transactions, handles comfortably without needing to shard for this event at all."*

---

### Pillar 5 — Communication & Articulation (Good to have)
Graded on: narrating section transitions out loud, explicitly naming the bottleneck before diving into schema, explaining *why not* (e.g., why not `SERIALIZABLE`, why not shard for 60K rows) rather than only *why*, and structuring the answer to visibly mirror the four-pillar rubric so an evaluator can tick boxes in real time.

---

## PART 3 — Feedback / Scoring Sheet

Fill during or immediately after the interview. Score 1–5 per pillar (1 = not addressed, 3 = addressed generically, 5 = addressed with Postgres-specific mechanism and staff-level tradeoff reasoning).

| Pillar | Weight | Score (1–5) | Postgres-specific evidence observed | Staff-level signal observed (tradeoffs, "why not," failure scenarios tied to this workload) |
|---|---|---|---|---|
| Data-Driven System Design | 25% | | | |
| Scalable Database Architecture | 25% | | | |
| System Resiliency & DR | 25% | | | |
| Building for Scale | 25% | | | |
| Communication & Articulation | Good to have | | | |

**Mandate compliance checklist (answer yes/no, include in written feedback verbatim):**
- [ ] Did the candidate engage with Postgres internals specifically (MVCC/bloat, isolation levels, replication modes, connection model) rather than treating Postgres as a generic "SQL database"?
- [ ] Did the candidate's depth match a staff-level bar — i.e. did they justify *why not* alternative approaches (sharding, `SERIALIZABLE`, synchronous-everywhere), not just describe what they'd do?
- [ ] Were all four core rubric pillars addressed, even if unevenly?
- [ ] Note any pillar that was skipped entirely or only superficially covered — this must be named explicitly in the written feedback, not just reflected in the numeric score.

**Written feedback template:**
> Overall: [1–2 sentence summary]
> Postgres depth: [specific mechanisms the candidate named correctly / got wrong / avoided]
> Staff-level bar: [met / partially met / not met] — [why]
> Strongest pillar: [ ] — [evidence]
> Weakest pillar: [ ] — [evidence, and whether it was asked about directly or the candidate never got there]

The funnel — explicitly framed as "keeping load off Postgres":

```
[Waiting room / token admission]   ← rate-limits entry, not DB-backed
        ↓
[Redis: seat availability + CAS hold]   ← absorbs ~99% of write attempts
        ↓ (only winners proceed)
[PgBouncer — transaction pooling]
        ↓
[Postgres primary — single-statement CAS UPDATE → orders insert]
        ↓ async replication
[Sync replica(s): durability for orders] + [Async replicas: seat-map reads]
```

**Closing line to say out loud (ties all four pillars together):**
*"The core scaling move is that Postgres only ever sees writes from users who already won a Redis-arbitrated hold — so a 3-million-way write storm at the edge becomes a few-thousand-QPS write load by the time it reaches the database. That's a number a well-tuned Postgres primary, with PgBouncer and short single-statement transactions, handles comfortably without needing to shard for this event at all."*

---

### Pillar 5 — Communication & Articulation (Good to have)
Graded on: narrating section transitions out loud, explicitly naming the bottleneck before diving into schema, explaining *why not* (e.g., why not `SERIALIZABLE`, why not shard for 60K rows) rather than only *why*, and structuring the answer to visibly mirror the four-pillar rubric so an evaluator can tick boxes in real time.

---

## PART 3 — Feedback / Scoring Sheet

Fill during or immediately after the interview. Score 1–5 per pillar (1 = not addressed, 3 = addressed generically, 5 = addressed with Postgres-specific mechanism and staff-level tradeoff reasoning).

| Pillar | Weight | Score (1–5) | Postgres-specific evidence observed | Staff-level signal observed (tradeoffs, "why not," failure scenarios tied to this workload) |
|---|---|---|---|---|
| Data-Driven System Design | 25% | | | |
| Scalable Database Architecture | 25% | | | |
| System Resiliency & DR | 25% | | | |
| Building for Scale | 25% | | | |
| Communication & Articulation | Good to have | | | |

**Mandate compliance checklist (answer yes/no, include in written feedback verbatim):**
- [ ] Did the candidate engage with Postgres internals specifically (MVCC/bloat, isolation levels, replication modes, connection model) rather than treating Postgres as a generic "SQL database"?
- [ ] Did the candidate's depth match a staff-level bar — i.e. did they justify *why not* alternative approaches (sharding, `SERIALIZABLE`, synchronous-everywhere), not just describe what they'd do?
- [ ] Were all four core rubric pillars addressed, even if unevenly?
- [ ] Note any pillar that was skipped entirely or only superficially covered — this must be named explicitly in the written feedback, not just reflected in the numeric score.

**Written feedback template:**
> Overall: [1–2 sentence summary]
> Postgres depth: [specific mechanisms the candidate named correctly / got wrong / avoided]
> Staff-level bar: [met / partially met / not met] — [why]
> Strongest pillar: [ ] — [evidence]
> Weakest pillar: [ ] — [evidence, and whether it was asked about directly or the candidate never got there]
