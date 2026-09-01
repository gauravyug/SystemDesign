# MLM Chain Sum Aggregation System Design

## Staff-Level System Design Interview

---

## 1. Problem Statement

Design a system to calculate the total sum of commissions/values in a multi-level marketing (MLM) chain structure. The system must efficiently:

- Calculate cumulative sum for any node (entire subtree commission)
- Handle updates to individual node values
- Support real-time queries with acceptable latency
- Scale to millions of users with deep hierarchies (100+ levels)
- Maintain consistency across distributed nodes
- Support rapid downline queries and performance reporting

---

## 2. Functional Requirements

| ID | API | Description |
|---|---|---|
| FR1 | `GetDownlineSum(userId)` | Return total commission sum for user and entire downline subtree |
| FR2 | `UpdateNodeValue(userId, value)` | Update commission value for a single user |
| FR3 | `AddToNetwork(parentId, newUserId)` | Add new user to network under parent |
| FR4 | `GetUserProfile(userId)` | Retrieve user info with current sum and rank |
| FR5 | `BulkUpdateValues(updates[])` | Batch update multiple node values |
| FR6 | `GetTreeStructure(userId, depth)` | Retrieve tree structure for visualization |
| FR7 | `GetTopPerformers(limit, timeWindow)` | Get top earners in past N days |
| FR8 | `AuditTrail(userId, timeWindow)` | Track all sum recalculations for compliance |

---

## 3. Non-Functional Requirements

| Category | Requirement | Rationale |
|---|---|---|
| **Latency** | GetDownlineSum: <100ms (p99) | Critical path for dashboard, app UI |
| **Latency** | UpdateNodeValue: <200ms (p99) | Commission updates throughout day |
| **Throughput** | 1M reads/sec, 100K writes/sec | Peak hours: morning payouts, evening metrics |
| **Availability** | 99.99% uptime (52.6 min/year) | Financial system, affects user earnings |
| **Consistency** | Eventual consistency OK | 5-10 second propagation acceptable |
| **Storage** | 10B nodes × 1KB average | 10TB primary, 10TB replicas |
| **Scalability** | Linear horizontal scaling | Add nodes without rebalancing entire tree |
| **Cost** | <$0.001 per user monthly | Competitive SaaS pricing |
| **Data Retention** | 7 years (regulatory) | Audit trail, compliance archival |
| **Network** | Geo-distributed | Sub-200ms latency across regions |

---

## 4. Core Entities & Data Models

### 4.1 User/Node Entity

```json
{
  "userId": "UUID",
  "parentId": "UUID (Parent in hierarchy)",
  "email": "String",
  "tier": "Enum (Bronze, Silver, Gold, Platinum)",
  "commissionValue": "Decimal (Current month commission)",
  "totalDownlineSum": "Decimal (Cached sum - eventual consistency)",
  "activeStatus": "Boolean",
  "joinDate": "Timestamp",
  "lastUpdated": "Timestamp",
  "metadata": {
    "region": "String",
    "phone": "String",
    "bankAccount": "String"
  }
}
```

### 4.2 Network Path Entity (for fast ancestor traversal)

```json
{
  "userId": "UUID",
  "ancestorPath": "[UUID]",
  "ancestorCount": "Int",
  "depth": "Int (Distance from root)"
}
```

### 4.3 Sum Update Log (for audit trail & recovery)

```json
{
  "logId": "UUID",
  "userId": "UUID",
  "previousSum": "Decimal",
  "newSum": "Decimal",
  "changeReason": "Enum (DirectUpdate, ChildUpdate, Recalculation)",
  "timestamp": "Timestamp",
  "calculationMs": "Int",
  "triggeredBy": "UUID (Admin or system)"
}
```

---

## 5. API Design

### 5.1 REST Endpoints

| Method | Endpoint | Type | Latency |
|---|---|---|---|
| GET | `/users/:userId/downline-sum` | Query (cacheable) | 100ms p99 |
| GET | `/users/:userId/profile` | Query (cacheable) | 50ms p99 |
| POST | `/users/:userId/commission` | Write (idempotent) | 200ms p99 |
| POST | `/users/:parentId/add-member` | Write (non-idempotent) | 300ms p99 |
| GET | `/reports/top-performers` | Query (cacheable, 1h TTL) | 200ms p99 |
| GET | `/users/:userId/tree?depth=3` | Query (light cache) | 150ms p99 |
| POST | `/admin/recalculate/:userId` | Admin heavy compute | 5s p99 |
| GET | `/audit/user/:userId?days=30` | Query (immutable) | 500ms p99 |

### 5.2 gRPC Internal Services

```protobuf
service SumAggregationService {
  // Get sum (cached, eventual consistency)
  rpc GetDownlineSum(GetSumRequest) 
    returns (SumResponse) {}
    
  // Invalidate cache, trigger recalc
  rpc InvalidateAncestorSums(InvalidateRequest) 
    returns (InvalidateResponse) {}
    
  // Batch get sums for performance
  rpc GetDownlineSumBatch(BatchRequest) 
    returns (BatchResponse) {}
    
  // Stream sum updates (for live dashboards)
  rpc StreamSumUpdates(StreamRequest) 
    returns (stream SumUpdate) {}
}
```

### 5.3 Request/Response Examples

```bash
GET /users/user-123/downline-sum
```

**Response:**
```json
{
  "userId": "user-123",
  "currentCommission": 15000,
  "totalDownlineSum": 450000,
  "childCount": 25,
  "leafCount": 200,
  "maxDepth": 8,
  "lastRecalculated": "2025-01-15T10:30:00Z",
  "isFresh": true
}
```

```bash
POST /users/user-123/commission
Body:
{
  "value": 15000,
  "period": "2025-01"
}
```

**Response:**
```json
{
  "success": true,
  "updatedAt": "2025-01-15T10:30:00Z",
  "ancestorsNotified": 25
}
```

---

## 6. High-Level System Architecture

### 6.1 Core Components

- **API Gateway**: Rate limiting, request routing, auth (OAuth2)
- **Sum Service**: Orchestrates queries, cache invalidation, recalculation
- **User Service**: User CRUD, profile, network topology
- **Event Queue**: Decouples updates, enables async processing
- **Cache Layer**: Redis for computed sums, user profiles
- **Primary DB**: PostgreSQL for user tree, audit logs
- **Replica DB**: Read-only for heavy queries, compliance
- **Message Queue**: RabbitMQ for async updates to ancestors
- **Search Index**: Elasticsearch for top performers, search
- **Audit Store**: Immutable log for compliance (S3 + DynamoDB)

### 6.2 Data Flow Diagram (High Level)

```
┌─────────────┐
│  Client App │
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│  API Gateway     │ (Auth, Rate limit)
└──────┬───────────┘
       │
    ┌──┴──────────────────────────┐
    │                             │
    ▼                             ▼
┌──────────────┐         ┌──────────────────┐
│  Sum Service │         │  User Service    │
└──┬────────┬──┘         └──┬────────┬──────┘
   │        │               │        │
   │   ┌────┴────────────────┘        │
   │   │                              │
   ▼   ▼                              ▼
┌─────────────────────────┐  ┌──────────────────┐
│  Cache (Redis)          │  │  Database Cluster│
│ - Sum cache (2h TTL)    │  │  - Primary (Write)
│ - Profile cache (4h TTL)│  │  - Replicas (Read)
└────────────┬────────────┘  └──────┬───────────┘
             │                      │
             └──────────┬───────────┘
                        │
                   ┌────▼────────┐
                   │Event Queue  │ (Kafka/RabbitMQ)
                   │- Value updates
                   │- Sum invalidations
                   │- Audit events
                   └─────────────┘
```

---

## 7. Detailed System Design with Advanced Components

### 7.1 Database Architecture

#### Primary Database: PostgreSQL (Partitioned by userId hash)

```sql
-- users (Partitioned: 16 shards)
users
├── userId (PK)
├── parentId (FK, indexed)
├── email
├── commissionValue
├── totalDownlineSum (materialized, cached in Redis)
├── lastSumRecalc (timestamp)
├── tier, active_status
└── Indexes:
    ├── idx_parent_id (for subtree queries)
    ├── idx_tier_period (for top performers)
    ├── idx_active_status (for audits)

-- paths (Denormalized ancestry)
paths
├── userId (PK)
├── ancestorPath (text[]) - [root, ..., parent]
├── depth (Int)
├── childCount (Int)
└── Indexes:
    ├── idx_userid
    ├── gin_ancestor_path (for contains queries)

-- sum_update_log (Immutable, partitioned by month)
sum_update_log
├── logId (PK)
├── userId (FK)
├── previousSum, newSum
├── changeReason
├── timestamp (Partition key)
├── triggeredBy
└── Indexes:
    ├── idx_userid_timestamp (for audit queries)
    ├── idx_timestamp (for compliance scans)

-- commission_ledger (Immutable, double-entry accounting)
commission_ledger
├── ledgerId (PK)
├── userId (FK)
├── transactionType (commission, payout, chargeback)
├── amount, currency
├── period (2025-01)
├── timestamp
└── Indexes:
    ├── idx_user_period
    ├── idx_timestamp
```

#### Replica Database: PostgreSQL Read-Only

Replicates from primary using logical replication with 2-5 second lag. Used for:
- Top performers queries (complex JOIN, GROUP BY)
- Historical audits (large scans)
- Compliance reports (heavy aggregations)
- Read-only backups for disaster recovery

### 7.2 Caching Strategy (Multi-Layer)

| Layer | Technology | Data | TTL | Impact |
|---|---|---|---|---|
| L1 Cache | In-memory (application) | Hot sums (top 10K users) | 5 min | Reduces load by 60% |
| L2 Cache | Redis | All user sums + profiles | 2h (sums), 4h (profiles) | Handles 1M reads/sec |
| L3 Cache | CDN | Public profiles, leaderboards | 1h | Geo-distributed latency <50ms |
| L4 DB | PostgreSQL | Single source of truth | N/A | Consistency guarantee |

**Cache Invalidation Strategy:**

- **Direct Invalidation**: When node value updated, invalidate node + all ancestors (up to 100 levels)
- **TTL**: 2-hour max TTL for sums (acceptable 5-10s drift)
- **Event-Driven**: RabbitMQ messages trigger async cache invalidation
- **Lazy Refresh**: If cache expired during query, return stale + trigger background refresh
- **Broadcast**: Publish to subscribers (WebSocket) for real-time dashboards

### 7.3 Event Queue Architecture (Message-Driven Updates)

**Message Broker: Apache Kafka (3 brokers, 16 partitions)**

**Topics:**

**1. commission.updates** (Partitioned by userId)
- Event: `{userId, oldValue, newValue, timestamp}`
- Consumers: Sum Service, Audit Service, Analytics
- Retention: 7 days
- SLA: <100ms end-to-end

**2. sum.invalidations** (Partitioned by userId)
- Event: `{userId, reason, initiatedBy}`
- Consumers: Cache Service, Real-time Service
- Retention: 24 hours
- SLA: <50ms propagation

**3. compliance.audit** (Immutable log)
- Event: `{logId, userId, previousSum, newSum}`
- Consumers: S3 archival, DynamoDB index
- Retention: 7 years
- SLA: Durability >99.9999%

**Consumer Groups:**
- `sum-calculator`: Recalculates ancestor sums
- `cache-invalidator`: Evicts cache keys
- `real-time-publisher`: Broadcasts to WebSocket clients
- `compliance-archiver`: Writes to S3 + DynamoDB
- `analytics-processor`: Aggregates daily metrics

**Processing Pipeline:**

1. User updates commission → Event published to Kafka
2. Sum Service consumes event → Invalidates Redis cache key for user
3. Cache-Invalidator also invalidates all ancestors (async)
4. Next query on user/ancestor checks Redis (miss) → reads from DB → caches result
5. Audit Service logs event to S3 (async), queryable via DynamoDB GSI

### 7.4 Data Consistency & Recalculation Service

**Challenge:** Eventual consistency + deep hierarchies = potential sum drift

#### 1. Full Recalculation (Monthly, off-peak)

```
├── BFS/DFS from roots to leaves
├── Recalc sum: sum(directCommission) + sum(childrenSums)
├── Compare cached vs computed
├── Log discrepancies to compliance DB
├── Update DB if drift > 0.01% (prevents rounding thrash)
├── Duration: 30 min (spread across night)
```

#### 2. Incremental Recalculation (On-Demand)

```
├── Triggered if user queries old cache
├── Recalc node only, don't bubble up
├── Bubble up if changed: update parent
├── Max propagation: 100 ancestors
├── Duration: <200ms (p99)
```

#### 3. Spot-Check Verification

```
├── Cron: Every 1 hour, random 1% of users
├── Verify: sum(children) + direct == cached
├── If drift, trigger full recalc for ancestor
├── Alert if systematic drift detected
```

#### 4. Conflict Resolution

```
├── If concurrent updates: Last-write-wins with timestamp
├── Record conflict in audit log
├── Manual review flag if value changed >50% in 1 day
```

### 7.5 Search Index for Analytics (Elasticsearch)

**Index: users_metrics** (Updated every 5 min via Kafka consumer)

**Mapping:**
```json
{
  "properties": {
    "userId": { "type": "keyword" },
    "email": { "type": "text", "analyzer": "email" },
    "tier": { "type": "keyword" },
    "region": { "type": "geo_point" },
    
    "currentCommission": { "type": "long" },
    "totalDownlineSum": { "type": "long" },
    "childCount": { "type": "integer" },
    
    "joinDate": { "type": "date" },
    "lastActivityDate": { "type": "date" },
    
    "30daySum": { "type": "long" },
    "90daySum": { "type": "long" }
  }
}
```

**Query Examples:**

**1. Top 100 earners last 30 days:**
```json
GET /users_metrics/_search
{
  "query": {
    "range": { "lastActivityDate": {"gte": "now-30d"}}
  },
  "sort": [{"30daySum": "desc"}],
  "size": 100
}
```

**2. Regional comparison:**
```json
GET /users_metrics/_search
{
  "aggs": {
    "by_region": {
      "terms": { "field": "region", "size": 50 },
      "aggs": {
        "avg_sum": { "avg": {"field": "totalDownlineSum"}}
      }
    }
  }
}
```

### 7.6 Compliance & Audit Store (S3 + DynamoDB)

#### S3 Archival (Immutable)

```
s3://audit-archive/logs/
├── 2025-01/user-123.jsonl  (append-only)
├── 2025-01/user-456.jsonl
├── MANIFEST.json
```

Features:
- Versioning: enabled
- Object Lock: Governance mode (7-year retention)
- Encryption: AES-256 (KMS)

#### DynamoDB Index (Queryable)

**Table: AuditIndex**
- PK: userId
- SK: timestamp (ISO8601)
- GSI1: timestamp (all logs, time-range queries)
- Attributes: userId, oldSum, newSum, changeReason, initiatedBy, approvalStatus, ipAddress, userAgent
- TTL: 7 years (automatic deletion)

**Compliance Queries:**

1. **Verify sum history:** `GetAuditTrail(userId, startDate, endDate)`
2. **Detect anomalies:** `GetChanges(reason=Chargeback)` last 90 days
3. **User disputes:** `GetAllCalculations(userId, period)`
4. **Regulatory report:** Export all logs for year Y as CSV

---

## 8. Scaling Strategy & Optimization Techniques

### 8.1 Problem: Deep Hierarchies (100+ levels)

Naive approach: DFS from leaf to root = O(depth) = O(100) = 100 lookups per query

**Solutions:**

1. **Cache sums at every node** → Solves with eventual consistency
2. **Denormalize paths table** → ancestorPath array → O(1) lookup
3. **Hybrid approach** → Cache first, fallback to DB

### 8.2 Problem: 1M reads/sec at 100ms p99 latency

| Technique | Implementation | Benefit |
|---|---|---|
| Caching | L1 in-app cache + Redis | Reduce DB from 1M to 100K QPS |
| Connection pool | 100 connections per app | Avoid connection exhaustion |
| Query optimization | idx_parent_id, idx_tier | Avoid full table scans |
| Replication | Read replicas for top performers | Distribute read load |
| Sharding | 16 shards by userId hash | Parallel query execution |
| Compression | LZ4 for cache payloads | Reduce network bandwidth |
| Batch APIs | GetDownlineSumBatch (up to 1K) | Amortize overhead |

### 8.3 Problem: 100K writes/sec, maintaining consistency

- **Write batching**: Buffer 100ms of updates, apply in micro-batches
- **WAL (Write-Ahead Logging)**: PostgreSQL ensures durability
- **Async propagation**: Updates queued to Kafka, consumed by cache invalidator
- **Idempotency**: Updates include requestId to prevent double-processing
- **Circuit breaker**: If ancestors take >100ms to invalidate, queue for async retry

### 8.4 Specific Optimizations

#### Optimization 1: Lazy Ancestor Path Calculation

```
When user joins (AddToNetwork):
├── parentId stored
├── Path calculated asynchronously
├── Cached for 1 week
├── Used for: Ancestor invalidation, compliance verification

Lookup example:
User1234 → parent=User999 → parent=User100 → parent=ROOT
ancestorPath = [ROOT, User100, User999, User1234]

Benefit: O(1) ancestor lookup instead of O(depth) parent traversals
```

#### Optimization 2: Buffered Ancestor Invalidation

```
When commission value updated:
├── Invalidate current node immediately
├── Queue ancestor invalidations (up to 100)
├── Batch within 5 seconds
├── Send to Kafka in single message

Example:
├── Update User1234 commission
├── Queue: Invalidate [User999, User100, ROOT]
├── Wait 5 sec, batch with other updates
├── Send: {users: [999, 100], timestamp: T}
├── Reduces message volume by 90%
```

#### Optimization 3: Smart Recalc on Read

```
When GetDownlineSum(userId) called:
├── Check Redis cache
├── If hit & fresh (age < 2 hours): return
├── If miss or stale:
│   ├── If depth < 5: Recalc on-demand (fast)
│   ├── If depth >= 5: Return cached value + schedule async recalc
│   └── Return within latency SLA
├── Background job recalcs high-depth nodes nightly

Benefit: Never blocks on deep recalculations
```

#### Optimization 4: Materialized Daily Snapshots

```
Nightly job:
├── Read all users from primary
├── Calculate exact sums (leaf to root)
├── Write to snapshot table (daily_sums_2025_01_15)
├── Compress & archive after 30 days

Benefit:
├── Compliance: Exact historical audit trail
├── Recovery: Can restore from snapshot
├── Analytics: Pre-computed for reports
├── Compliance queries: <100ms on immutable snapshot
```

---

## 9. Operational Concerns & Monitoring

### 9.1 Key Metrics & Alerting

| Metric | Component | Target | Alert Threshold |
|---|---|---|---|
| Latency (p99) | GetDownlineSum | <100ms | Alert if >150ms |
| Error Rate | API endpoints | <0.01% | Alert if >0.05% |
| Cache Hit Ratio | Redis | >95% | Alert if <90% |
| Sum Drift | Cached vs actual | <0.01% | Alert if >0.5% |
| Queue Backlog | Kafka consumer lag | <5s | Alert if >30s |
| DB Replication Lag | Primary→Replica | <5s | Alert if >10s |
| Recalc Duration | Full monthly recalc | <30 min | Alert if >1 hour |
| Audit Log Volume | Events/sec | 100 events/sec | Alert if >500 |
| Storage Growth | Database size | 1TB/month | Alert if accelerating |

### 9.2 Disaster Recovery & High Availability

- **Multi-region deployment**: Active-passive with automatic failover
- **Database**: Primary in US-East, replica in US-West + EU
- **Cache**: Replicated Redis (master-slave), <5s sync
- **Kafka**: Replication factor = 3, min.insync.replicas = 2
- **Backups**: Hourly snapshots to S3, geo-replicated, 7-year retention
- **RTO**: 5 minutes (automated failover)
- **RPO**: <1 minute

### 9.3 Security Considerations

- **Authentication**: OAuth2 (Google, Microsoft), 2FA for admins
- **Authorization**: Role-based access control (RBAC) - users see own + downline only
- **Encryption**: TLS 1.3 in-transit, AES-256 at-rest (KMS)
- **Audit**: All sum changes logged with user ID, IP, timestamp
- **Rate limiting**: 100 req/s per user, 1000 req/s per IP
- **DDoS protection**: CloudFlare, automatic throttling on traffic spike

### 9.4 Testing Strategy

| Test Type | Scope | Target | Tools |
|---|---|---|---|
| Unit Tests | Sum calculation logic | >90% coverage | Jest/Go testing |
| Integration Tests | API → DB → Cache | Critical paths | Testcontainers |
| Load Tests | 1M reads/sec, 100K writes/sec | Sustained 1 hour | K6/JMeter |
| Chaos Engineering | Fail Kafka consumer, kill DB | Verify graceful degradation | Chaos Monkey |
| Compliance Tests | Audit trail integrity | Immutability verified | Custom scripts |
| E2E Tests | Full MLM workflows | Add user, update, query | Selenium/API |

---

## 10. Architecture Summary & Future Evolution

### Architecture Layers

| Layer | Technology | Responsibility |
|---|---|---|
| Presentation | Web/Mobile app | Dashboard, leaderboards, profile |
| API | REST (public) + gRPC (internal) | Endpoints, authentication, rate limit |
| Business Logic | Sum Service, Recalc Service | Computation, caching, invalidation |
| Data Layer | PostgreSQL primary, replicas | Consistency, audit trail |
| Cache Layer | Redis + CDN | Speed, geo-distribution |
| Message Queue | Kafka | Async updates, decoupling |
| Search | Elasticsearch | Analytics, reporting |
| Compliance | S3 + DynamoDB | Immutable audit, 7-year retention |

### Future Improvements (Phase 2+)

- **Graph Database (Neo4j)**: More efficient tree queries if hierarchy becomes more complex
- **Stream Processing (Flink)**: Real-time anomaly detection on commission patterns
- **ML Models**: Predict high performers, fraud detection on unusual hierarchies
- **Multi-currency**: Support 50+ currencies with FX volatility handling
- **Sharding Rebalancing**: Dynamic shard migration based on user density
- **TimeScale DB**: Specialized time-series for commission trends
- **WebSocket API**: Real-time sum updates for live dashboards

### Cost Breakdown (Annual, 10M users)

| Component | Specification | Cost |
|---|---|---|
| Compute (API servers) | 20 servers × $100/mo | $24,000 |
| Database (RDS PostgreSQL) | db.r6i.4xlarge × 4 | $120,000 |
| Cache (Redis) | ElastiCache 500GB × 3 | $36,000 |
| Message Queue (Kafka) | Managed (Confluent) | $48,000 |
| Search (Elasticsearch) | 7-node cluster, 100GB | $28,000 |
| S3 (Audit + backups) | 100TB storage | $2,400 |
| Data Transfer | Inter-region replication | $18,000 |
| Monitoring (DataDog) | Per-host, logs, APM | $36,000 |
| **Total Monthly** | | **$26,250** |
| **Per User Annual** | | **$0.0315** |

---

## Summary

This MLM Chain Sum System is designed for:

✅ **Scale**: 10B nodes, 1M reads/sec, 100K writes/sec  
✅ **Performance**: <100ms p99 latency for queries  
✅ **Reliability**: 99.99% uptime with disaster recovery  
✅ **Consistency**: Eventual consistency with <10s propagation  
✅ **Compliance**: 7-year immutable audit trail  
✅ **Cost**: $0.03 per user annually at scale  

The architecture uses proven patterns (caching, event-driven async, partitioning, denormalization) to handle the unique challenges of MLM hierarchies while maintaining financial data integrity.
