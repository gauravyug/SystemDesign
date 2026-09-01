# Staff-Level Distributed Systems & Microservices Interview Q&A

This document captures the detailed interview discussion around multi-tenancy, large-file ingestion, microservices, API security, privacy/compliance, distributed logging, caching, SQL performance, indexing, .NET parallelism, queues/topics, failure recovery, and related Staff-level design considerations.

---

## 1. Multi-Tenant Architecture: How Do You Design It Safely?

### Question
How would you design a multi-tenant system so that multiple customers can share infrastructure while their data and workloads remain isolated?

### Answer
A multi-tenant application shares some infrastructure among customers (tenants), but tenant isolation must be treated as a system invariant across authentication, authorization, databases, caches, queues, workers, and observability.

The basic principle is:

> **Share compute where possible, but make tenant isolation an invariant across identity, data, caching, and asynchronous processing.**

Tenant identity should come from a trusted authenticated identity, such as a JWT claim, rather than an arbitrary client-provided header or query parameter.

A trusted `TenantContext` should be established early in request processing and propagated throughout synchronous and asynchronous work.

### Data isolation models

Common approaches include:

1. Shared database and shared tables with a `tenant_id` column.
2. Shared database with schema-per-tenant.
3. Database-per-tenant.
4. Hybrid architecture where ordinary tenants share infrastructure while large/regulatory tenants receive dedicated resources.

Example shared table:

```sql
CREATE TABLE Payments (
    PaymentId UNIQUEIDENTIFIER,
    TenantId UNIQUEIDENTIFIER NOT NULL,
    Amount DECIMAL(18,2),
    Status VARCHAR(30),
    PRIMARY KEY (TenantId, PaymentId)
);
```

Every access must be tenant scoped:

```sql
SELECT *
FROM Payments
WHERE TenantId = @TenantId
  AND PaymentId = @PaymentId;
```

Defense in depth should look like:

```text
Authentication
    ↓
Trusted TenantContext
    ↓
Authorization
    ↓
Tenant-scoped Repository / ORM
    ↓
Database Row-Level Security
```

For asynchronous processing, every message should contain trusted tenant context and resource identifiers:

```text
TenantId
JobId / EventId
EntityId
TraceId
```

Consumers should still verify tenant/resource ownership before accessing data.

### Staff-level consideration

Tenant isolation has multiple dimensions:

- Security isolation
- Data isolation
- Performance isolation
- Failure isolation

A system can be secure from cross-tenant reads but still suffer from poor performance isolation if one tenant consumes most workers, DB connections, or queue capacity.

---

## 2. How Do You Prevent Cross-Tenant Data Leakage From Query Bugs?

### Question
Suppose a developer forgets to include `TenantId` in a query. How do you prevent one customer's data from being returned to another customer?

### Answer
Do not rely on developers remembering to write `WHERE TenantId = ...` everywhere.

Use defense in depth.

First, derive tenant identity from authentication rather than trusting request input.

Second, expose tenant-scoped repository APIs instead of unrestricted database access.

Avoid:

```csharp
GetPayment(paymentId)
```

Prefer something conceptually like:

```csharp
var repository = new TenantRepository(tenantContext);
var payment = await repository.GetPaymentAsync(paymentId);
```

Third, use database Row-Level Security where appropriate. The database session can carry the current tenant and enforce a policy such that even an accidental:

```sql
SELECT * FROM Payments;
```

only returns rows belonging to the current tenant.

Cache keys must also include tenant identity:

```text
Bad:
payment:123

Good:
tenant:A:payment:123
```

Integration tests should intentionally attempt cross-tenant reads.

For highly sensitive customers, stronger physical isolation such as separate schemas, credentials, or databases may be appropriate.

---

## 3. One Tenant Uploads a 20 GB File: How Do You Prevent a Noisy Neighbor?

### Question
Hundreds of tenants upload files, but one tenant uploads a 20 GB file and starts consuming 80% of the system. How would you prevent that tenant from affecting everybody else?

### Answer
Do not process the entire 20 GB file as one queue message or one giant operation.

The architecture should separate upload, scheduling, and processing:

```text
Tenant
  ↓
Object Storage
  ↓
Import Metadata
  ↓
Tenant-Aware Scheduler
  ↓
Bounded Chunks
  ↓
Worker Pool
  ↓
Database
```

The file should be stored in object/blob storage. The queue contains metadata rather than the actual file.

Break the file into bounded chunks, for example 50 MB, 100 MB, or a fixed number of records.

Then use tenant-aware scheduling:

```text
A1 → B1 → C1 → A2 → B2 → C2 ...
```

instead of:

```text
A1 → A2 → A3 → A4 → ... → A200
```

Per-tenant quotas can control:

- Concurrent chunks
- Worker concurrency
- DB connections
- DB write throughput
- Network bandwidth
- CPU/memory
- Queue depth

A weighted fair scheduling strategy such as weighted round robin or deficit round robin can be used.

A useful model is **guaranteed share + opportunistic bursting**. A large tenant can use spare capacity while the system is idle, but once other tenants require resources, it is throttled back to its fair allocation.

Backpressure should also be based on downstream health. If database CPU, transaction log, p99 latency, or lock contention rises, reduce dispatch/concurrency rather than continuing to overload the database.

---

## 4. How Would You Find a Bottleneck Affecting Only One Tenant?

### Question
The system looks healthy overall, but one tenant says their imports are very slow. How would you diagnose that?

### Answer
Tenant-aware observability is critical because fleet-wide averages can hide a problem affecting one customer.

Instrument each pipeline stage:

```text
Upload
  ↓
Queue Wait
  ↓
File Read / Parsing
  ↓
Validation
  ↓
Database Read/Write
  ↓
Completion
```

Track metrics by tenant such as:

```text
processing_latency_ms
queue_wait_ms
db_write_latency_ms
records_processed_per_sec
error_rate
worker_concurrency
retry_rate
```

Compare the affected tenant's p95/p99 with peer tenants and the fleet baseline.

Distributed traces should include:

```text
TenantId
JobId
TraceId
```

If parsing and validation are normal but database upsert takes 12 seconds, the bottleneck is likely in the DB path.

Large tenants can expose scale-dependent problems such as:

- Missing indexes
- Huge tenant partitions
- Bad execution plans
- Parameter sensitivity
- Lock contention
- Hot shards
- Quota saturation
- Excessive retries

Also inspect tenant-specific saturation: active workers versus quota, DB connections versus limit, queue backlog, and resource entitlement.

---

## 5. Efficiently Processing 10 Million Records in .NET

### Question
A file contains 10 million records. Some need to be inserted and others updated. How would you process it efficiently in .NET?

### Answer
Avoid row-by-row database processing such as:

```text
SELECT row
IF exists → UPDATE
ELSE → INSERT
```

for every record. That produces millions of database round trips.

Use a bulk ingestion pipeline:

```text
Object Storage
    ↓
Streaming Parser (.NET)
    ↓
Batches
    ↓
Staging Table
    ↓
Set-Based UPDATE / INSERT
    ↓
Target Table
```

The .NET process should stream the file instead of loading all 10 million rows into memory.

Use batches, perhaps 10K–100K rows depending on row size and workload.

For SQL Server, `SqlBulkCopy` is a good option:

```csharp
using var bulkCopy = new SqlBulkCopy(connection);
bulkCopy.DestinationTableName = "dbo.ImportStaging";
bulkCopy.BatchSize = 50_000;
bulkCopy.BulkCopyTimeout = 0;
await bulkCopy.WriteToServerAsync(dataReader);
```

A staging table can contain:

```text
JobId
TenantId
RecordId
Business Fields
RowHash
```

Then perform set-based updates:

```sql
UPDATE T
SET T.Name = S.Name,
    T.Amount = S.Amount
FROM Target T
JOIN ImportStaging S
  ON T.TenantId = S.TenantId
 AND T.RecordId = S.RecordId
WHERE S.JobId = @JobId;
```

And insert missing records:

```sql
INSERT INTO Target (TenantId, RecordId, Name, Amount)
SELECT S.TenantId, S.RecordId, S.Name, S.Amount
FROM ImportStaging S
WHERE S.JobId = @JobId
  AND NOT EXISTS (
      SELECT 1
      FROM Target T
      WHERE T.TenantId = S.TenantId
        AND T.RecordId = S.RecordId
  );
```

Prefer explicit UPDATE + INSERT when it provides clearer concurrency semantics than blindly relying on `MERGE`.

A critical index might be:

```text
(TenantId, RecordId)
```

Do not use one transaction for all 10 million rows. Commit bounded batches to reduce lock duration, transaction-log pressure, rollback cost, and recovery time.

If possible, skip unchanged rows using a row hash/version.

---

## 6. What Happens If Processing Fails at 70%?

### Question
Suppose the 10-million-row import crashes after processing 70%. Do you restart everything?

### Answer
No. Design processing around **checkpointed, idempotent batches**.

Example:

```text
Batch 1   COMPLETED
Batch 2   COMPLETED
...
Batch 159 COMPLETED
Batch 160 PROCESSING ← worker crashes
Batch 161 PENDING
```

When another worker resumes the job, it retries batch 160 rather than restarting the entire file.

Ideally, data changes and checkpoint status are committed atomically:

```text
BEGIN TRANSACTION
    Apply batch changes
    Mark batch COMPLETED
COMMIT
```

If the worker dies before commit, both changes are rolled back and the batch can safely retry.

Queue systems usually provide at-least-once delivery, so consumers should be idempotent. Unique constraints, idempotency keys, and deterministic upsert logic help prevent duplicate effects.

For long-running work, use queue leases/visibility timeouts appropriately. If a worker crashes, the lease eventually expires and another worker can pick up the work.

---

## 7. Full Multi-Tenant Bank Import Architecture

### Question
Design a system where hundreds of bank customers upload huge files, traffic can spike 20x, records need inserts/updates, one tenant must not impact another, failures must be recoverable, and customers need completion notifications.

### Answer
A suitable high-level architecture is:

```text
                         ┌─────────────────┐
Tenant ──HTTPS──────────►│ API Gateway/WAF │
                         └────────┬────────┘
                                  │
                                  v
                         ┌─────────────────┐
                         │ Import Service  │
                         └───────┬─────────┘
                                 │ pre-signed upload URL
                                 v
                         ┌─────────────────┐
                         │ Object Storage  │
                         └───────┬─────────┘
                                 │
                                 v
                         ┌─────────────────┐
                         │ ImportJob DB    │
                         └───────┬─────────┘
                                 │
                                 v
                         ┌─────────────────┐
                         │ Durable Queue   │
                         └───────┬─────────┘
                                 │
                                 v
                     ┌──────────────────────┐
                     │ Tenant-Aware         │
                     │ Scheduler            │
                     └─────────┬────────────┘
                               │
                               v
                     ┌──────────────────────┐
                     │ Autoscaled Workers   │
                     └─────────┬────────────┘
                               │
                               v
                     ┌──────────────────────┐
                     │ Staging Tables       │
                     └─────────┬────────────┘
                               │
                               v
                     ┌──────────────────────┐
                     │ Production Database  │
                     └─────────┬────────────┘
                               │
                               │ ImportCompleted
                               v
                            Topic
                         /    |     \
                 Notification Audit Analytics
```

The application should not proxy a 20 GB upload through its web servers. The control plane creates an import and returns a pre-signed object-storage upload URL.

The ImportJob database is the durable source of truth. Queue messages only indicate work that needs processing.

Example job lifecycle:

```text
UPLOADING
   ↓
QUEUED
   ↓
PROCESSING
   ↓
COMPLETED
```

or:

```text
FAILED / PARTIAL / CANCELLED
```

A 20x traffic burst is absorbed by object storage and the durable queue. Ingress can temporarily exceed processing capacity.

Worker autoscaling should consider queue depth and oldest-message age but must be capped by downstream-safe database capacity.

Conceptually:

```text
desired workers = min(queue-driven demand, downstream-safe capacity)
```

Completion notifications should be asynchronous. A notification failure should not change a successfully completed import into a failed import.

---

## 8. How Do You Decide Which Bank Job Gets Priority?

### Question
If many bank tenants submit work simultaneously, how do you decide which job runs first?

### Answer
Do not use a single hardcoded customer priority.

Priority can consider:

- Business criticality
- SLA/deadline
- Waiting time
- Customer tier
- Workload cost
- Recent resource consumption

Example lanes:

```text
P0 → regulatory / settlement / fraud / cutoff-critical
P1 → time-sensitive customer operations
P2 → normal daily imports
P3 → backfills / reprocessing / analytics
```

Use aging so low-priority work cannot starve forever:

```text
effective_priority = base_priority + waiting_time_factor
```

For deadline-aware scheduling:

```text
slack_time = deadline - current_time - estimated_processing_time
```

Jobs with smaller slack have higher urgency.

The Staff-level principle is:

> **Priority determines urgency; fairness determines entitlement.**

Priority decisions should also be auditable.

---

## 9. Push Queue vs Pull Queue

### Question
Would you use a push queue or a pull queue for file processing?

### Answer
For expensive, variable-duration processing, pull is usually preferable because workers control consumption.

With a pull model:

```text
Worker → Queue: Give me work
Queue  → Worker: Batch 123
```

If the database becomes overloaded, worker concurrency can be reduced and messages remain safely queued.

Amazon SQS is primarily pull-based. A worker receives a message, which becomes invisible for a visibility-timeout period. On success the worker deletes it. If the worker crashes, the timeout expires and the message becomes available again.

For a 20 GB operation, do not depend on one message being invisible for hours. Prefer bounded chunks or periodically renew the visibility timeout.

A useful distinction is:

> **Pull controls how fast I consume; tenant-aware scheduling controls whose work I consume.**

---

## 10. Queue vs Topic

### Question
What is the difference between a queue and a topic, and where would you use each?

### Answer
A queue represents point-to-point work distribution. Multiple workers compete for messages, but one work item is normally handled by one consumer.

```text
ProcessImport command
        ↓
      Queue
     /  |  \
Worker Worker Worker
```

A topic represents publish/subscribe. One event can be independently consumed by multiple downstream systems.

```text
ImportCompleted
       ↓
     Topic
   /   |    \
Audit Notify Analytics
```

A useful rule:

> **Queue → “Please DO this.”**
>
> **Topic → “This HAPPENED.”**

---

## 11. Thread Safety When Processing Multiple Tenants in .NET

### Question
If multiple threads process multiple tenants simultaneously, how do you make the code thread-safe?

### Answer
The first rule is to avoid shared mutable state.

Never use a global/static tenant variable:

```csharp
// Dangerous
CurrentTenantId = tenantId;
```

Two concurrent operations could overwrite it and potentially cause cross-tenant leakage.

Tenant context should be immutable and scoped to the operation, preferably passed explicitly.

`DbContext` is not thread-safe. Never share one EF `DbContext` across concurrent workers. Each worker should receive its own DI scope, connection/context, transaction, and repositories.

Example:

```csharp
await Parallel.ForEachAsync(messages, async (msg, ct) =>
{
    await using var scope = serviceProvider.CreateAsyncScope();
    var processor = scope.ServiceProvider
                         .GetRequiredService<ImportProcessor>();

    await processor.ProcessAsync(msg, ct);
});
```

For shared in-memory maps, use thread-safe structures such as `ConcurrentDictionary`, but remember that compound operations can still require explicit synchronization.

A local per-tenant `SemaphoreSlim` can limit concurrency:

```csharp
var limiter = tenantLimiters.GetOrAdd(
    tenantId,
    _ => new SemaphoreSlim(5));

await limiter.WaitAsync(ct);
try
{
    await ProcessBatchAsync(batch, ct);
}
finally
{
    limiter.Release();
}
```

However, an in-process semaphore only coordinates one process. Multiple Kubernetes pods require database/distributed concurrency control.

For competing updates to the same row, optimistic concurrency can be used:

```sql
UPDATE Account
SET Amount = @NewAmount,
    Version = Version + 1
WHERE TenantId = @TenantId
  AND AccountId = @AccountId
  AND Version = @ExpectedVersion;
```

If zero rows are updated, another writer changed the row first.

Key distinction:

> **Thread safety protects memory inside one process; concurrency control protects correctness across the distributed system.**

---

## 12. Which Parallelism Approach Would You Use in Modern .NET?

### Question
There are many ways to start parallel work in .NET. Which approach would you normally use today and why?

### Answer
For a straightforward known collection of independent asynchronous work, `Parallel.ForEachAsync` is a strong default.

```csharp
await Parallel.ForEachAsync(
    batches,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 8,
        CancellationToken = ct
    },
    async (batch, token) =>
    {
        await ProcessBatchAsync(batch, token);
    });
```

It provides bounded concurrency, natural `async/await`, cancellation support, and lets the .NET runtime manage thread-pool scheduling.

Do not normally create one raw `Thread` per job. That is unnecessarily low-level.

Also avoid creating an enormous number of tasks and immediately doing `Task.WhenAll` if that results in unbounded work and downstream overload.

For a sustained producer-consumer pipeline that requires explicit buffering and backpressure, `Channel<T>` is useful:

```csharp
var channel = Channel.CreateBounded<ImportBatch>(100);
```

A fixed worker pool consumes from it. A bounded channel prevents producers from filling memory indefinitely.

Practical rule:

```text
Parallel.ForEachAsync
    → known collection + bounded async parallelism

Channel<T>
    → long-running pipeline + buffering + backpressure
```

For this import system, most work is I/O-bound: object storage reads, database operations, and network waits. Use true asynchronous APIs rather than creating physical threads just to wait on I/O.

---

## 13. Can We Use `Parallel.ForEach` Instead of Channels?

### Question
Can we just use `Parallel.ForEach`? I have used that before but haven't used `Channel<T>`.

### Answer
Yes. Do not claim production experience with `Channel<T>` if you have not used it.

For synchronous CPU-bound work:

```csharp
Parallel.ForEach(
    batches,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 8
    },
    batch =>
    {
        ProcessBatch(batch);
    });
```

For asynchronous I/O-heavy processing, prefer:

```csharp
await Parallel.ForEachAsync(
    batches,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 8,
        CancellationToken = ct
    },
    async (batch, token) =>
    {
        await ProcessBatchAsync(batch, token);
    });
```

Avoid blocking asynchronous operations inside `Parallel.ForEach`:

```csharp
Parallel.ForEach(batches, batch =>
{
    ProcessBatchAsync(batch).Wait(); // Avoid
});
```

A good interview response if asked about Channels is:

> “I haven't used `Channel<T>` directly in production, but I understand the producer-consumer pattern and how bounded channels provide buffering and backpressure. In systems I've worked on I've generally used `Parallel.ForEach` or `Parallel.ForEachAsync` with controlled concurrency. For this workload I'd start with `Parallel.ForEachAsync`; if I needed a continuously running pipeline with explicit buffering and backpressure, I'd evaluate `Channel<T>`.”

---

## 14. How Do You Choose Microservice Boundaries?

### Question
Some applications are monoliths and are being split into microservices. How do you decide service boundaries, and how do you avoid creating a distributed monolith?

### Answer
Do not split a monolith according to technical layers.

This is usually a bad split:

```text
Controller Service
       ↓
Business Service
       ↓
Repository Service
       ↓
Database
```

It simply turns local method calls into network calls.

Instead, split around **business capabilities / bounded contexts**.

For example:

```text
Upload / Import Service
Processing Service
Account Service
Notification Service
Audit Service
```

A service should ideally have:

- A cohesive business responsibility
- Clear data ownership
- A well-defined API/event contract
- Independent scaling requirements where appropriate
- The ability to evolve and deploy largely independently

A strong boundary heuristic is:

> **Things that change together should stay together.**

If two proposed services require coordinated deployments, share transactions constantly, and synchronously call each other on almost every request, they may actually belong together.

Different scaling/reliability characteristics can indicate good boundaries. File processing may need massive burst scaling, whereas notifications have a completely different workload.

### Avoid shared databases

Avoid:

```text
Service A ─┐
Service B ─┼── Shared Tables
Service C ─┘
```

Prefer explicit ownership:

```text
Service A → Data A
Service B → Data B
Service C → Data C
```

Even if databases initially share the same physical SQL Server, logical ownership should be clear.

Other services should use APIs or events rather than querying another service's private tables.

### Avoid long synchronous chains

Avoid architectures where every operation requires:

```text
A → B → C → D → E
```

because latency and failure dependencies accumulate.

Use events where immediate consistency is unnecessary.

### Avoid distributed transactions

If almost every operation requires atomic commits across three service databases, the service boundaries may be wrong.

Use local transactions and, where appropriate, patterns such as outbox, idempotent consumers, and sagas.

A modular monolith can be a better intermediate step than prematurely creating dozens of tightly coupled services.

Key principle:

> **High cohesion inside a service; low coupling between services.**

---

## 15. How Would You Protect a Customer-Facing API?

### Question
After splitting the monolith into microservices, you expose APIs to customers. How do you ensure only authorized callers can access sensitive data?

### Answer
Protect the API in layers:

```text
Customer
   ↓
API Gateway / WAF
   ↓
Authentication
   ↓
Authorization
   ↓
Tenant / Resource Authorization
   ↓
Data Access Policy
   ↓
Database
```

For customer APIs, OAuth 2.0 / OIDC is normally preferable to custom authentication.

A signed access token might contain:

```json
{
  "sub": "user-123",
  "tenant_id": "bank-A",
  "scope": "accounts.read payments.read",
  "aud": "payments-api",
  "exp": 1780000000
}
```

The API validates:

- Signature
- Issuer
- Audience
- Expiration
- Required scopes/claims

Authentication answers:

> Who are you?

Authorization answers:

> Are you allowed to perform this operation on this specific resource?

For:

```text
GET /accounts/987
```

the system must check that the authenticated tenant is actually allowed to access account 987.

Never trust:

```text
?tenantId=TenantB
```

as the authority for tenant identity.

The database query should still be tenant scoped:

```sql
SELECT ...
FROM Accounts
WHERE TenantId = @AuthenticatedTenantId
  AND AccountId = @AccountId;
```

At the edge, use TLS, WAF/DDoS protection, request validation, payload limits, and per-client/tenant rate limits.

For service-to-service calls, use workload/service identities, OAuth client credentials, managed identity, or mTLS rather than shared hardcoded passwords.

Apply least privilege.

Audit sensitive operations without logging secrets or sensitive payloads.

---

## 16. Privacy, Integrity, and Compliance Beyond Authentication

### Question
Besides authentication and authorization, what architectural components do you need for privacy, integrity, and compliance?

### Answer
Security must continue after a request passes authentication.

Important controls include:

### Encryption

Use TLS for data in transit and encryption for databases/object storage at rest. Particularly sensitive fields may require application-level encryption.

### Key management

Use centralized KMS/HSM capabilities, key rotation, separation of duties, and avoid hardcoded keys.

### Integrity

For uploaded files, calculate and verify checksums/hashes:

```text
Upload File
   ↓
Calculate Checksum
   ↓
Store Object + Expected Checksum
   ↓
Worker Downloads/Streams
   ↓
Verify Integrity
   ↓
Process
```

Use database constraints, idempotency keys, and tamper-resistant records where appropriate.

### Data minimization

A service should only receive/store fields it actually requires. Avoid unnecessary replication of sensitive data.

### Masking/tokenization

Mask or tokenize account/card/PII data, particularly in logs, support systems, and analytics.

### Retention/deletion

Define how long data must be retained for regulatory/business requirements and securely remove it after the retention period where applicable.

### Secrets management

Passwords, API keys, certificates, and credentials belong in a secret-management system and should be rotated.

### Network segmentation

Databases and internal services should not be directly internet-accessible.

### Backup and disaster recovery

Backups should be encrypted and restoration should be tested. Define RPO/RTO requirements.

### Monitoring

Detect anomalous behavior such as unusual bulk exports, repeated failures, or unexpected access patterns.

### Compliance evidence

Controls should be measurable and auditable: access reviews, deployment history, vulnerability scans, patching, and change records.

---

## 17. How Do You Design Logging for a Distributed System?

### Question
How would you design logging/auditing in a distributed microservices architecture?

### Answer
A single business request can cross many components:

```text
Customer
   ↓
API Gateway
   ↓
Import Service
   ↓
Queue
   ↓
Processing Service
   ↓
Database
   ↓
Notification Service
```

Independent plain-text logs are not enough because it becomes difficult to determine which entries belong to the same operation.

Introduce a **Trace ID / Correlation ID** at the entry point and propagate it through HTTP requests, queue messages, and events.

```text
traceId = abc-123
tenantId = Tenant-A
jobId = import-789
```

All downstream components carry the same trace ID.

Use structured logs:

```csharp
_logger.LogInformation(
    "Processing import {ImportId} for tenant {TenantId}",
    importId,
    tenantId);
```

Conceptually the resulting log contains:

```json
{
  "timestamp": "...",
  "level": "Information",
  "service": "ImportProcessor",
  "traceId": "abc-123",
  "tenantId": "tenant-A",
  "importId": "import-789",
  "event": "ImportStarted"
}
```

Send logs from all instances to centralized storage/search.

Then an operator can search by:

```text
traceId = abc-123
```

or:

```text
tenantId = A AND importId = 789
```

### Operational logging vs auditing

Operational logs answer:

> What happened and why did the system fail?

Examples:

```text
DB timeout
queue retry
worker restart
exception
latency
```

Audit logs answer:

> Who did what to which resource and when?

Example:

```json
{
  "actor": "user-123",
  "tenant": "bank-A",
  "action": "ImportUploaded",
  "resource": "import-789",
  "timestamp": "...",
  "result": "Success"
}
```

Banking audit logs should have stronger protections: append-only/tamper-resistant storage, restricted access, retention controls, and careful handling of sensitive fields.

Do not log raw PII, credentials, tokens, account numbers, or complete financial payloads unless there is an explicit, controlled requirement.

### Logs, metrics, traces, audit

A useful interview framework is:

```text
Logs    → What happened?
Metrics → How much / how often?
Traces  → Where did it happen and where was time spent?
Audit   → Who did what?
```

Distributed tracing can show:

```text
Trace abc-123

API Gateway              30 ms
 └─ Import Service       20 ms
     └─ Processing      820 ms
         ├─ Blob Read   100 ms
         ├─ Validation   70 ms
         └─ DB Write    630 ms
```

---

## 18. Cache Eviction and Cache Invalidation

### Question
How do you handle cache invalidation? Should you use LRU, LFU, or both for transaction data?

### Answer
First distinguish **eviction** from **invalidation**.

LRU/LFU answer:

> Which entry should be removed when cache capacity is exhausted?

Invalidation answers:

> How do I ensure cached data does not remain stale after the underlying database changes?

They solve different problems.

### Cache-aside pattern

```text
Client
  ↓
Service
  ├── Redis Cache
  └── Database
```

Read path:

```text
1. Check cache
2. Hit  → return cached value
3. Miss → read DB
4. Store result with TTL
5. Return
```

Update path:

```text
1. Update database
2. Commit transaction
3. Invalidate corresponding cache entry
```

Next read misses the cache and reloads fresh data.

### LRU

Least Recently Used removes data that has not been accessed recently.

```text
T1 last accessed 2 hours ago
T2 last accessed 20 minutes ago
T3 last accessed 2 seconds ago

LRU prefers T1 for eviction.
```

### LFU

Least Frequently Used prefers entries with low access frequency.

```text
T1 accessed 2 times
T2 accessed 5000 times
T3 accessed 50 times

LFU prefers T1.
```

Do not normally implement both manually. A distributed cache such as Redis can provide eviction policies. The correct policy depends on workload characteristics.

### Mutable transaction state

Be careful caching highly mutable financial state.

For example:

```text
PENDING → COMPLETED
```

Returning stale `PENDING` for a long time may be undesirable.

Use a combination of:

- Short TTL
- Explicit invalidation
- Event-driven invalidation where appropriate
- Versioning when needed

Example:

```text
Payment Service
     ↓ PaymentUpdated
Event Bus
     ↓
Cache Invalidator
     ↓
DELETE payment:123
```

Also ensure cache keys are tenant scoped:

```text
tenant:A:payment:123
```

Do not cache all old transactions merely because they are old. Cache entries that have sufficient reuse or are expensive to retrieve.

Key point:

> **LRU/LFU are capacity-management policies; TTL/invalidation/versioning are data-freshness mechanisms.**

---

## 19. How Do You Diagnose and Fix a Slow SQL Query?

### Question
A SQL query is running slowly in production. Walk through how you would find the problem and fix it.

### Answer
Do not immediately assume an index is missing. Diagnose systematically.

### Step 1: Measure the problem

Collect:

- Duration
- CPU time
- Logical reads
- Physical reads where relevant
- Execution frequency
- Waits
- Parameter values/workload characteristics

Determine whether the query is always slow or slow only for particular tenants/parameters.

### Step 2: Inspect the actual execution plan

Look for:

- Table/index scans on large datasets
- Expensive joins
- Repeated key lookups
- Sorts
- Hash spills / tempdb spills
- Implicit conversions
- Bad cardinality estimates
- Inefficient parallelism
- Missing/ineffective indexes

Example:

```text
Index Scan: 8,000,000 rows
       ↓
Filter
       ↓
Return: 20 rows
```

For:

```sql
SELECT *
FROM Transactions
WHERE TenantId = @TenantId
  AND AccountId = @AccountId;
```

an appropriate index may allow an index seek:

```sql
CREATE INDEX IX_Transactions_Tenant_Account
ON Transactions(TenantId, AccountId);
```

### Step 3: Inspect query formulation

Avoid non-sargable predicates such as:

```sql
WHERE YEAR(TransactionDate) = 2026
```

Prefer:

```sql
WHERE TransactionDate >= '2026-01-01'
  AND TransactionDate <  '2027-01-01'
```

This allows the optimizer to use a date index more efficiently.

### Step 4: Compare estimated vs actual rows

If the plan estimates:

```text
100 rows
```

but actually processes:

```text
5,000,000 rows
```

it may select a poor join algorithm or memory grant.

Investigate:

- Stale statistics
- Data skew
- Parameter sensitivity/sniffing

Multi-tenancy makes this especially relevant:

```text
Tenant A → 100 rows
Tenant B → 20 million rows
```

A plan optimized for Tenant A may perform badly for Tenant B.

### Step 5: Determine whether it is actually blocking

A query might consume only 50 ms of CPU but take 20 seconds wall-clock because it is waiting for a lock.

Inspect:

- Wait statistics
- Blocking chains
- Long-running transactions
- Deadlocks
- Isolation levels

Key phrase:

> **Execution plan tells me how SQL is executing the query; wait statistics tell me what it is waiting for.**

### Step 6: Fix and verify

Potential fixes include:

- Rewrite query
- Add/change index
- Remove unnecessary columns/rows
- Update statistics
- Address parameter-sensitive plans
- Reduce blocking/transaction scope
- Fix schema/data-access patterns

Always benchmark before and after:

```text
Before
Duration      8 sec
Logical reads 2,000,000
CPU           4 sec

After
Duration      80 ms
Logical reads 2,500
CPU           20 ms
```

---

## 20. How Do You Choose an Index, and What Are the Problems With Over-Indexing?

### Question
How do you design the right index for a query, and what problems can too many indexes cause?

### Answer
Indexes should be driven by actual query access patterns rather than indexing every column.

Consider:

```sql
SELECT TransactionId, Amount, Status
FROM Transactions
WHERE TenantId = @TenantId
  AND AccountId = @AccountId
  AND TransactionDate >= @FromDate;
```

A possible composite index is:

```sql
CREATE INDEX IX_Transactions_Tenant_Account_Date
ON Transactions
(
    TenantId,
    AccountId,
    TransactionDate
)
INCLUDE
(
    TransactionId,
    Amount,
    Status
);
```

The key columns help SQL Server locate rows efficiently. Included columns can make the index covering so SQL Server does not need additional key lookups to retrieve the selected fields.

Column order matters. Equality predicates are often useful before range predicates, but actual selectivity and workload patterns must also be considered.

### Problems with over-indexing

Indexes improve reads but add write cost.

If a table has many indexes:

```text
INSERT
  ├─ Base table
  ├─ Index 1
  ├─ Index 2
  ├─ Index 3
  ├─ Index 4
  └─ Index 5
```

SQL Server must maintain the affected index structures whenever data changes.

Consequences include:

- Slower INSERT operations
- Slower UPDATE operations
- Slower DELETE operations
- More storage consumption
- More buffer-pool/cache pressure
- More transaction-log and I/O work
- Longer index maintenance/rebuild operations
- Redundant/overlapping indexes
- Additional statistics and operational maintenance

This matters especially for the 10-million-row bulk-import workload discussed earlier. A target table with many unnecessary indexes can make ingestion substantially more expensive.

A strong principle is:

> **Indexing is a read/write tradeoff: indexes reduce read cost but increase write and maintenance cost.**

Periodically identify unused, duplicate, and overlapping indexes and retain the smallest set of indexes that provides meaningful workload value.

---

# Interview Cheat Sheet

## Multi-Tenancy

> Tenant identity comes from trusted authentication, is propagated through every layer, and is enforced again at the data layer.

## Noisy Neighbor

> Use tenant-aware scheduling, per-tenant quotas, fair-share allocation, bounded chunks, and backpressure.

## Large Imports

> Stream → batch → bulk load staging → set-based DB operations → checkpoint.

## Failure Recovery

> At-least-once delivery + idempotent processing + durable checkpoints.

## Microservice Boundaries

> High cohesion within a service, low coupling between services. Things that change together should stay together.

## API Security

> Authentication establishes identity; authorization decides whether that identity may perform this operation on this resource.

## Distributed Logging

> Structured centralized logs + trace/correlation IDs propagated across HTTP, queues, and events.

## Observability

> Logs tell what happened; metrics tell how much/how often; traces tell where it happened; audit tells who did what.

## Caching

> LRU/LFU decide what to evict. TTL/invalidation/versioning decide how to prevent stale data.

## Slow SQL

> Measure → actual execution plan → cardinality → waits/blocking → query/index/statistics fix → benchmark.

## Indexing

> Design indexes from access patterns. Every additional index improves some reads at the cost of writes, storage, memory, and maintenance.

## .NET Parallelism

> Prefer async I/O with bounded concurrency. Use `Parallel.ForEachAsync` for straightforward parallel processing and consider `Channel<T>` for sustained producer-consumer pipelines requiring explicit backpressure.

## Queue vs Topic

> Queue = “do this.” Topic = “this happened.”

## Staff-Level Design Theme

> Do not optimize individual components in isolation. Protect system-wide correctness, tenant isolation, downstream capacity, recoverability, observability, and operability.
