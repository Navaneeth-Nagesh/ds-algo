# Chapter 15: System Design Building Blocks

[← Previous: SD Fundamentals](14-system-design-fundamentals.md) | [Next: Distributed Systems →](16-distributed-systems.md)

---

## 15.1 Load Balancing

### Types of Load Balancers

```
Layer 4 (Transport):
  - Routes based on IP + port
  - Doesn't inspect content
  - Very fast, low overhead
  - Examples: HAProxy (TCP mode), AWS NLB

Layer 7 (Application):
  - Routes based on HTTP headers, URL, cookies
  - Can do content-based routing, SSL termination
  - More features, slightly slower
  - Examples: Nginx, HAProxy (HTTP mode), AWS ALB, Envoy
```

### Load Balancing Algorithms

```python
# 1. Round Robin — Simple rotation
class RoundRobin:
    def __init__(self, servers):
        self.servers = servers
        self.index = 0

    def next(self):
        server = self.servers[self.index]
        self.index = (self.index + 1) % len(self.servers)
        return server

# 2. Weighted Round Robin — Servers with different capacities
class WeightedRoundRobin:
    def __init__(self, servers_weights):
        # servers_weights: [("s1", 5), ("s2", 3), ("s3", 2)]
        self.servers = []
        for server, weight in servers_weights:
            self.servers.extend([server] * weight)
        self.index = 0

    def next(self):
        server = self.servers[self.index]
        self.index = (self.index + 1) % len(self.servers)
        return server

# 3. Least Connections — Route to least busy server
class LeastConnections:
    def __init__(self, servers):
        self.connections = {s: 0 for s in servers}

    def next(self):
        server = min(self.connections, key=self.connections.get)
        self.connections[server] += 1
        return server

    def release(self, server):
        self.connections[server] -= 1

# 4. IP Hashing — Sticky sessions
def ip_hash(client_ip, servers):
    idx = hash(client_ip) % len(servers)
    return servers[idx]

# 5. Least Response Time
# Route to server with fastest response + fewest active connections

# 6. Random — Simple and surprisingly effective at scale
import random
def random_lb(servers):
    return random.choice(servers)

# 7. Power of Two Random Choices
# Pick 2 random servers, choose the less loaded one
# Much better than pure random, almost as good as least connections
def power_of_two(servers, load_fn):
    a, b = random.sample(servers, 2)
    return a if load_fn(a) <= load_fn(b) else b
```

### Algorithm Comparison

```
Algorithm              │ Simplicity │ Balance │ Sticky │ Scenario
───────────────────────│────────────│─────────│────────│─────────────
Round Robin            │ ★★★★★     │ ★★★☆☆  │ No     │ Equal servers
Weighted Round Robin   │ ★★★★☆     │ ★★★★☆  │ No     │ Unequal capacity
Least Connections      │ ★★★☆☆     │ ★★★★★  │ No     │ Variable request times
IP Hash                │ ★★★★★     │ ★★★☆☆  │ Yes    │ Session affinity
Least Response Time    │ ★★☆☆☆     │ ★★★★★  │ No     │ Heterogeneous servers
Random                 │ ★★★★★     │ ★★★☆☆  │ No     │ Large server pools
Power of 2 Choices     │ ★★★★☆     │ ★★★★★  │ No     │ Best general purpose
```

### Health Checks

```
Active Health Checks:
  LB periodically pings servers (HTTP, TCP, custom)
  Unhealthy servers removed from rotation

  Config example:
    interval: 10s
    timeout: 5s
    unhealthy_threshold: 3 consecutive failures
    healthy_threshold: 2 consecutive successes

Passive Health Checks:
  LB monitors actual traffic responses
  Server marked unhealthy on error spike
```

---

## 15.2 Caching

### Cache Architecture

```
Client → CDN (edge cache) → API Gateway cache →
Application cache (in-memory) → Distributed cache (Redis) → Database

Multiple layers, each progressively closer to source of truth.
```

### Caching Strategies

```
1. Cache-Aside (Lazy Loading):
   App checks cache → miss → reads DB → writes to cache

   ┌─────┐    miss    ┌────┐
   │Cache │←──────────│App │
   └─────┘    ───→    └────┘
              write      │ read
                        ┌────┐
                        │ DB │
                        └────┘

   ✓ Simple, only caches what's needed
   ✗ Cache miss = 3 round trips (check cache, read DB, write cache)
   ✗ Data can become stale

2. Read-Through:
   Cache itself loads from DB on miss
   App only talks to cache

   ✓ Simpler application code
   ✗ Cache library needs DB connector

3. Write-Through:
   App writes to cache → cache writes to DB (synchronous)

   ✓ Cache always up to date
   ✗ Higher write latency (2 writes)
   ✗ Caches data that may never be read

4. Write-Behind (Write-Back):
   App writes to cache → cache async writes to DB (batched)

   ✓ Very fast writes
   ✓ Batch DB writes reduce load
   ✗ Data loss risk if cache fails before flush
   ✗ Complex

5. Write-Around:
   App writes directly to DB, bypassing cache
   Cache populated only on read miss

   ✓ Avoids caching data that won't be re-read
   ✗ Recent writes always cause cache miss
```

### Cache Eviction Policies

```python
from collections import OrderedDict
from threading import Lock

# LRU Cache (Least Recently Used) — Most common
class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = Lock()

    def get(self, key):
        with self.lock:
            if key not in self.cache:
                return None
            self.cache.move_to_end(key)  # Mark as recently used
            return self.cache[key]

    def put(self, key, value):
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
            self.cache[key] = value
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)  # Remove least recent


# LFU Cache (Least Frequently Used)
class LFUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}          # key → value
        self.freq = {}           # key → frequency
        self.freq_keys = {}      # frequency → OrderedDict of keys
        self.min_freq = 0

    def _update_freq(self, key):
        f = self.freq[key]
        self.freq[key] = f + 1
        del self.freq_keys[f][key]
        if not self.freq_keys[f]:
            del self.freq_keys[f]
            if self.min_freq == f:
                self.min_freq += 1
        self.freq_keys.setdefault(f + 1, OrderedDict())[key] = None

    def get(self, key):
        if key not in self.cache:
            return -1
        self._update_freq(key)
        return self.cache[key]

    def put(self, key, value):
        if self.capacity <= 0:
            return
        if key in self.cache:
            self.cache[key] = value
            self._update_freq(key)
            return
        if len(self.cache) >= self.capacity:
            # Evict LFU (tie-break by LRU)
            evict_key, _ = self.freq_keys[self.min_freq].popitem(last=False)
            del self.cache[evict_key]
            del self.freq[evict_key]
        self.cache[key] = value
        self.freq[key] = 1
        self.min_freq = 1
        self.freq_keys.setdefault(1, OrderedDict())[key] = None
```

### Eviction Policy Comparison

```
Policy     │ Mechanism                      │ Use Case
───────────│────────────────────────────────│──────────────────
LRU        │ Remove least recently used     │ General purpose (best default)
LFU        │ Remove least frequently used   │ Popularity-based (CDNs)
FIFO       │ Remove first inserted          │ Simple, time-based
Random     │ Remove random entry            │ Low overhead
TTL        │ Expire after time              │ Session data, tokens
ARC        │ Adaptive (LRU + LFU)           │ Self-tuning (ZFS uses this)
```

### Cache Invalidation

```
Pattern                │ How                          │ Tradeoff
───────────────────────│──────────────────────────────│──────────
Time-Based (TTL)       │ Auto-expire after N seconds  │ Stale during TTL
Event-Driven           │ Invalidate on data change    │ Complex writes
Version-Based          │ Key includes version number  │ Needs versioning
Pub/Sub                │ Publish invalidation events  │ Infra complexity

Cache Stampede Prevention:
  Problem: Cache expires → 1000 requests simultaneously hit DB
  Solutions:
    1. Locking: Only one request fetches, others wait
    2. Probabilistic early expiry: Renew before TTL
    3. Background refresh: Async refresh before expiry
    4. Stale-while-revalidate: Serve stale, refresh in background
```

### Distributed Cache (Redis)

```
Redis Architecture Options:

1. Standalone: Single instance
   - Simple, low latency
   - Single point of failure

2. Sentinel: Auto failover
   ┌──────────┐
   │ Sentinel  │ (monitors + auto-failover)
   └──────────┘
   ┌──────┐    ┌─────────┐    ┌─────────┐
   │Master │───→│Replica 1│───→│Replica 2│
   └──────┘    └─────────┘    └─────────┘

3. Cluster: Sharded, horizontal scaling
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
   │ Master   │  │ Master   │  │ Master   │
   │ Replica  │  │ Replica  │  │ Replica  │
   └─────────┘  └─────────┘  └─────────┘

   16384 hash slots distributed across shards
   Automatic resharding when nodes added/removed

Redis Data Structures for System Design:
  String:      Simple cache, counters, rate limiting
  Hash:        User profiles, session data
  List:        Message queues, activity feeds
  Set:         Unique items, tags, mutual friends
  Sorted Set:  Leaderboards, priority queues, time-series
  HyperLogLog: Unique visitor counting (12KB for billions)
  Bloom Filter: Membership testing (via RedisBloom)
  Stream:      Event logs, message queues (like Kafka lite)
  Geo:         Location-based features
```

---

## 15.3 Content Delivery Network (CDN)

```
CDN Architecture:

  User (Tokyo) ──→ [Edge Server Tokyo] ──→ [Origin Server US]
  User (London) ──→ [Edge Server London] ──→ [Origin Server US]
  User (NYC) ──→ [Edge Server NYC] ──→ [Origin Server US]

  First request: Edge fetches from origin → caches locally
  Subsequent requests: Served from edge (fast!)

Pull CDN:
  - Content pulled from origin on first request
  - Cached at edge after first miss
  - ✓ No push needed, ✓ Automatic
  - ✗ First request slow (cache miss)
  - Best for: Large catalogs, unpredictable access

Push CDN:
  - Content proactively pushed to edges
  - ✓ Always fast, no cold misses
  - ✗ Need to manage what's pushed
  - ✗ Storage cost at every edge
  - Best for: Video streaming, known popular content

CDN Use Cases:
  Static assets:    JS, CSS, images, fonts
  Video streaming:  HLS/DASH segments
  API responses:    Cacheable endpoints
  Dynamic at edge:  Edge computing (Cloudflare Workers, Lambda@Edge)

Common CDNs: Cloudflare, Akamai, AWS CloudFront, Fastly, Google Cloud CDN
```

---

## 15.4 Message Queues & Event Streaming

### Message Queue Pattern

```
Producer → [Queue] → Consumer

Properties:
  - Asynchronous: Producer doesn't wait for consumer
  - Decoupling: Producer and consumer are independent
  - Buffering: Handle burst traffic
  - Guaranteed delivery: At-least-once or exactly-once

Point-to-Point: One consumer processes each message
  Producer → [Queue] → Consumer 1  (messages distributed)
                     → Consumer 2

Pub/Sub: All subscribers get all messages
  Publisher → [Topic] → Subscriber 1  (each gets a copy)
                      → Subscriber 2
                      → Subscriber 3
```

### Message Queue Comparison

```
System     │ Model      │ Throughput │ Ordering    │ Use Case
───────────│────────────│────────────│─────────────│──────────────
RabbitMQ   │ Queue      │ 10K/s     │ Per-queue   │ Task queues, RPC
Kafka      │ Log        │ 1M+/s    │ Per-partition│ Event streaming, logs
SQS        │ Queue      │ Unlimited │ FIFO option │ AWS microservices
Redis Pub/Sub│Pub/Sub   │ 100K/s   │ None        │ Real-time notifications
NATS       │ Pub/Sub    │ 10M+/s   │ Per-subject │ IoT, microservices
Pulsar     │ Log+Queue  │ 1M+/s    │ Per-topic   │ Multi-tenancy
```

### Apache Kafka Deep Dive

```
Kafka Architecture:

  Producers → [Topic: user-events]          → Consumers
               ├── Partition 0: [msg1, msg4, msg7]  → Consumer Group A
               ├── Partition 1: [msg2, msg5, msg8]  → Consumer Group A
               └── Partition 2: [msg3, msg6, msg9]  → Consumer Group A

Key Concepts:
  Topic:     Category of messages (like a table)
  Partition: Ordered, immutable log within a topic
  Offset:    Position of message in partition
  Consumer Group: Set of consumers sharing work on a topic
  Broker:    Kafka server that stores partitions

Guarantees:
  Ordering:   Messages ordered within a partition (not across partitions)
  Durability: Replicated across brokers (replication factor)
  Retention:  Time-based or size-based (days, weeks, forever)
  Delivery:
    At-most-once:   Fire and forget
    At-least-once:  Retry until ACK (may duplicate)
    Exactly-once:   Idempotent producer + transactional consumer

When to Use Kafka:
  ✓ Event-driven architecture    ✓ Real-time analytics
  ✓ Log aggregation              ✓ Change data capture (CDC)
  ✓ Stream processing            ✓ Audit trails
```

### Event-Driven Architecture

```
Event Sourcing:
  Store sequence of events, not current state

  Traditional: UPDATE account SET balance = 900 WHERE id = 1

  Event Sourced:
    Event 1: AccountCreated {id: 1, balance: 1000}
    Event 2: MoneyWithdrawn {id: 1, amount: 100}
    Event 3: MoneyDeposited {id: 1, amount: 50}
    Current state = replay all events → balance = 950

  ✓ Complete audit trail
  ✓ Temporal queries ("what was state at time T?")
  ✓ Event replay for debugging/recovery
  ✗ Complex reads (need CQRS)
  ✗ Storage grows forever

CQRS (Command Query Responsibility Segregation):
  ┌─────────────┐
  │ Write Model  │──→ [Event Store] ──→ [Read Model]
  │ (commands)   │                      │ (queries)  │
  │ Normalized   │                      │Denormalized│
  └─────────────┘                      └────────────┘

  Separate models for reading and writing
  Write: Optimized for consistency and business rules
  Read: Optimized for query performance (denormalized views)

  ✓ Independent scaling of reads vs writes
  ✓ Each model optimized for its purpose
  ✗ Eventual consistency between models
  ✗ Complexity
```

---

## 15.5 Database Systems

### SQL vs NoSQL Decision Guide

```
Choose SQL When:                    Choose NoSQL When:
  ✓ Complex queries/joins           ✓ Simple lookups by key
  ✓ ACID transactions needed        ✓ Massive scale (millions QPS)
  ✓ Well-defined schema             ✓ Schema flexibility needed
  ✓ Relational data                 ✓ High write throughput
  ✓ Complex aggregations            ✓ Geographic distribution
  ✓ Data integrity critical         ✓ Real-time analytics

SQL Databases:         NoSQL Types:
  PostgreSQL             Key-Value: Redis, DynamoDB
  MySQL                  Document: MongoDB, Couchbase
  Oracle                 Wide-Column: Cassandra, HBase
  SQL Server             Graph: Neo4j, Amazon Neptune
  CockroachDB            Time-Series: InfluxDB, TimescaleDB
  Vitess (sharded MySQL) Search: Elasticsearch, Solr
```

### Database Scaling

```
Read Scaling:
  1. Read replicas (separate R/W traffic)
  2. Caching layer (Redis, Memcached)
  3. CDN for static content
  4. Materialized views / denormalization

Write Scaling:
  1. Vertical scaling (bigger machine)
  2. Sharding (horizontal partitioning)
  3. Write-behind caching
  4. Message queue buffering (batch writes)
  5. CQRS (separate write model)

Connection Scaling:
  1. Connection pooling (PgBouncer, ProxySQL)
  2. Connection multiplexing
  3. Serverless databases (Aurora Serverless)
```

### Database Indexing

```
B-Tree Index (Default):
  - Balanced tree, O(log n) lookups
  - Good for: range queries, equality, sorting
  - Most common index type

Hash Index:
  - O(1) lookups
  - Good for: exact match only
  - Not for: range queries, sorting

Composite Index:
  - Index on multiple columns
  - Order matters! (a, b, c) supports queries on (a), (a,b), (a,b,c)
  - Leftmost prefix rule

Covering Index:
  - Index contains all columns needed for query
  - No table lookup needed (index-only scan)

Full-Text Index:
  - Inverted index for text search
  - Supports stemming, stop words, ranking

Partial Index:
  - Index only rows matching a condition
  - Saves space, faster updates

When NOT to index:
  - Small tables (sequential scan is fine)
  - High-write, low-read tables (index maintenance is expensive)
  - Columns with very low cardinality (boolean)
```

### SQL Query Optimization

```
EXPLAIN ANALYZE plan reading:
  Seq Scan:     Full table scan (usually bad for large tables)
  Index Scan:   Uses index, fetches from table
  Index Only:   All data from index (best)
  Bitmap Scan:  Good for medium selectivity
  Nested Loop:  OK for small inner table
  Hash Join:    Good for large tables, equality
  Merge Join:   Good for large sorted tables

Common Optimizations:
  1. Use covering indexes
  2. Avoid SELECT * (reduce I/O)
  3. Limit results (pagination)
  4. Denormalize for read-heavy workloads
  5. Partition large tables
  6. Use connection pooling
  7. Batch inserts/updates
  8. Use prepared statements
```

---

## 15.6 Storage Systems

### Object Storage

```
Properties:
  - Flat namespace (no hierarchy, though "/" simulates folders)
  - Immutable objects (versioning for updates)
  - HTTP/REST access
  - Massive scale (exabytes)

Examples: AWS S3, Google Cloud Storage, Azure Blob Storage

Use Cases:
  - Static files (images, videos, documents)
  - Backups and archives
  - Data lake (analytics)
  - Static website hosting

S3 Consistency Model:
  - Strong read-after-write consistency (since 2020)
  - List operations eventually consistent
```

### Block vs File vs Object Storage

```
Type    │ Access     │ Performance │ Scale    │ Use Case
────────│────────────│─────────────│──────────│──────────────────
Block   │ Low-level  │ Highest     │ Limited  │ Databases, VMs (EBS)
File    │ File/path  │ Medium      │ Medium   │ Shared files (EFS, NFS)
Object  │ HTTP/REST  │ High        │ Unlimited│ Media, backups (S3)
```

### Blob Storage Design

```
Large File Upload:
  1. Client requests pre-signed upload URL
  2. Client uploads directly to object store (bypasses API server)
  3. On completion, client notifies API server
  4. API server records metadata in database

Chunked Upload (for large files):
  1. Client splits file into chunks (e.g., 5MB each)
  2. Each chunk uploaded independently (parallel, resumable)
  3. Server reassembles chunks
  4. Support for pause/resume

Optimization:
  - Content-addressed storage (hash-based deduplication)
  - Delta sync (only upload changed chunks) — Dropbox uses this
  - Client-side compression before upload
```

---

## 15.7 Rate Limiting

### Algorithms

```python
import time
from collections import defaultdict

# 1. Token Bucket — Smooth, allows bursts
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate          # tokens added per second
        self.capacity = capacity  # max tokens
        self.tokens = capacity
        self.last_refill = time.time()

    def allow(self, tokens=1):
        now = time.time()
        # Refill tokens
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
        self.last_refill = now
        # Check
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False

# 2. Sliding Window Log — Exact count
class SlidingWindowLog:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests = defaultdict(list)

    def allow(self, user_id):
        now = time.time()
        cutoff = now - self.window
        # Remove old requests
        self.requests[user_id] = [
            t for t in self.requests[user_id] if t > cutoff
        ]
        if len(self.requests[user_id]) < self.max_requests:
            self.requests[user_id].append(now)
            return True
        return False

# 3. Fixed Window Counter — Simple
class FixedWindowCounter:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window = window_seconds
        self.counters = {}

    def allow(self, user_id):
        now = time.time()
        window_key = int(now // self.window)
        key = (user_id, window_key)
        count = self.counters.get(key, 0)
        if count < self.max_requests:
            self.counters[key] = count + 1
            return True
        return False

# 4. Sliding Window Counter — Hybrid (most practical)
class SlidingWindowCounter:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window = window_seconds
        self.prev_counts = {}
        self.curr_counts = {}
        self.prev_window = {}

    def allow(self, user_id):
        now = time.time()
        curr_window = int(now // self.window)

        if self.prev_window.get(user_id) != curr_window - 1:
            self.prev_counts[user_id] = self.curr_counts.get(user_id, 0)
            self.curr_counts[user_id] = 0
            self.prev_window[user_id] = curr_window - 1

        # Weighted count from previous window
        elapsed_in_window = now % self.window
        weight = 1 - (elapsed_in_window / self.window)
        count = (self.prev_counts.get(user_id, 0) * weight +
                 self.curr_counts.get(user_id, 0))

        if count < self.max_requests:
            self.curr_counts[user_id] = self.curr_counts.get(user_id, 0) + 1
            return True
        return False
```

### Rate Limiting Comparison

```
Algorithm            │ Memory   │ Accuracy │ Burst │ Use Case
─────────────────────│──────────│──────────│───────│────────────────
Token Bucket         │ O(1)     │ Good     │ Yes   │ API gateways (most common)
Leaky Bucket         │ O(1)     │ Good     │ No    │ Traffic shaping
Fixed Window         │ O(1)     │ Approx   │ Edge  │ Simple counting
Sliding Window Log   │ O(N)     │ Exact    │ No    │ Strict rate limiting
Sliding Window Count │ O(1)     │ Good     │ Edges │ Best balance
```

### Distributed Rate Limiting

```
Options:
  1. Redis-based: INCR + EXPIRE (atomic via Lua script)
  2. Token bucket in Redis: DECR token count
  3. Rate limit at API Gateway (AWS API Gateway, Kong, Envoy)

Redis Lua script for sliding window:
  local key = KEYS[1]
  local window = tonumber(ARGV[1])
  local max_requests = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])

  redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
  local count = redis.call('ZCARD', key)
  if count < max_requests then
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('EXPIRE', key, window)
    return 1
  end
  return 0

Strategies for distributed systems:
  - Centralized: All rate limit checks go through Redis
    + Accurate, - Redis is bottleneck
  - Local + Sync: Each instance tracks locally, periodically syncs
    + Fast, - Slightly inaccurate during sync gap
  - Sticky routing: Same user → same instance (IP hash)
    + Simple, - Uneven distribution
```

---

## 15.8 Service Discovery & Configuration

### Service Discovery

```
Client-side Discovery:
  ┌────────┐    query    ┌──────────┐
  │ Client  │───────────→│ Registry │
  └────────┘             └──────────┘
       │                    ↑ register
       │ direct call    ┌───────┐
       └───────────────→│Service│
                        └───────┘

  Examples: Netflix Eureka + Ribbon
  ✓ Client chooses instance (custom LB logic)
  ✗ Client coupled to registry

Server-side Discovery:
  ┌────────┐         ┌────────┐    query    ┌──────────┐
  │ Client  │────────→│  LB    │───────────→│ Registry │
  └────────┘         └────────┘             └──────────┘
                        │ route            ↑ register
                     ┌───────┐
                     │Service│
                     └───────┘

  Examples: AWS ALB, Kubernetes Service, Consul + Envoy
  ✓ Simpler clients
  ✗ LB is extra hop

Popular Tools:
  Consul:      Service mesh, KV store, health checks
  etcd:        Distributed KV store (Kubernetes uses this)
  ZooKeeper:   Configuration, naming, synchronization
  Kubernetes:  Built-in service discovery via DNS + Services
```

### Circuit Breaker Pattern

```python
import time
from enum import Enum

class State(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing, reject requests
    HALF_OPEN = "half_open" # Testing recovery

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30,
                 success_threshold=3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.state = State.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == State.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = State.HALF_OPEN
                self.success_count = 0
            else:
                raise Exception("Circuit is OPEN — request rejected")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        self.failure_count = 0
        if self.state == State.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = State.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = State.OPEN

# State diagram:
#   CLOSED ──(failures > threshold)──→ OPEN
#     ↑                                  │
#     │                              (timeout)
#     │                                  ↓
#     └──(successes > threshold)──── HALF_OPEN
```

### Retry & Backoff Strategies

```python
import time
import random

# Exponential Backoff with Jitter
def retry_with_backoff(func, max_retries=5, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise e
            # Exponential backoff: 1s, 2s, 4s, 8s, 16s
            delay = base_delay * (2 ** attempt)
            # Add jitter to prevent thundering herd
            jitter = random.uniform(0, delay * 0.5)
            time.sleep(delay + jitter)

# Strategies:
# Fixed:        Always wait same time
# Linear:       1s, 2s, 3s, 4s, 5s
# Exponential:  1s, 2s, 4s, 8s, 16s
# Exp + Jitter: 1s±0.5s, 2s±1s, 4s±2s (RECOMMENDED)
# Decorrelated: min(cap, random(base, prev_delay * 3))
```

---

## 15.9 Search Systems

### Inverted Index

```
Documents:
  Doc1: "the cat sat on the mat"
  Doc2: "the dog sat on the log"
  Doc3: "cats and dogs"

Inverted Index:
  Term   │ Document IDs (Posting List)
  ───────│───────────────────────────
  the    │ [Doc1:2, Doc2:2]   (frequency)
  cat    │ [Doc1:1, Doc3:1]
  sat    │ [Doc1:1, Doc2:1]
  on     │ [Doc1:1, Doc2:1]
  mat    │ [Doc1:1]
  dog    │ [Doc2:1, Doc3:1]
  log    │ [Doc2:1]
  and    │ [Doc3:1]

Search "cat sat":
  cat posting list: [Doc1, Doc3]
  sat posting list: [Doc1, Doc2]
  Intersection: [Doc1] ← matches both terms
```

### Elasticsearch Architecture

```
Cluster:
  ┌─────────────────────────────────────────┐
  │ Node 1 (Master)                         │
  │  ┌────────────┐  ┌────────────┐         │
  │  │ Shard 0    │  │ Shard 1    │         │
  │  │ (Primary)  │  │ (Replica)  │         │
  │  └────────────┘  └────────────┘         │
  │                                         │
  │ Node 2                                  │
  │  ┌────────────┐  ┌────────────┐         │
  │  │ Shard 1    │  │ Shard 2    │         │
  │  │ (Primary)  │  │ (Replica)  │         │
  │  └────────────┘  └────────────┘         │
  │                                         │
  │ Node 3                                  │
  │  ┌────────────┐  ┌────────────┐         │
  │  │ Shard 2    │  │ Shard 0    │         │
  │  │ (Primary)  │  │ (Replica)  │         │
  │  └────────────┘  └────────────┘         │
  └─────────────────────────────────────────┘

Lucene Inside Each Shard:
  - Segment files (immutable)
  - Inverted index per segment
  - Segment merging in background
  - Near-real-time search (~1 second delay)

Search Flow:
  1. Query hits coordinating node
  2. Scatter to all relevant shards
  3. Each shard searches locally (inverted index)
  4. Gather, merge, rank results
  5. Return top-K results
```

### Search Relevance

```
TF-IDF (Term Frequency × Inverse Document Frequency):
  TF = frequency of term in document / total terms in document
  IDF = log(total documents / documents containing term)
  Score = TF × IDF

  Common words (the, is) → low IDF → low score
  Rare relevant words → high IDF → high score

BM25 (Better, used by Elasticsearch):
  - Improved version of TF-IDF
  - Handles term saturation (TF doesn't grow linearly)
  - Considers document length normalization

  score(D, Q) = Σ IDF(qi) × (f(qi,D) × (k1+1)) / (f(qi,D) + k1 × (1 - b + b × |D|/avgdl))
  k1 ≈ 1.2, b ≈ 0.75

Boosting strategies:
  - Field boost (title > body)
  - Recency boost (newer > older)
  - Popularity boost (more views > fewer)
  - Personalization boost
```

---

## 15.10 Real-Time Communication

### WebSocket

```
Connection Lifecycle:
  1. HTTP Upgrade request
  2. Server responds with 101 Switching Protocols
  3. Full-duplex communication on same TCP connection
  4. Either side can close

  Client ────── HTTP Upgrade ──────→ Server
  Client ←───── 101 Switch ─────── Server
  Client ←────→ Full Duplex ←───→ Server

Use Cases:
  - Chat applications
  - Live sports/stock updates
  - Collaborative editing (Google Docs)
  - Gaming
  - Live notifications

Scaling WebSockets:
  Problem: WebSocket connections are stateful (sticky)

  Solutions:
    1. Sticky sessions (IP hash) at load balancer
    2. Redis Pub/Sub for cross-server messages
    3. Dedicated WebSocket servers

    Server 1 ←──→ [Redis Pub/Sub] ←──→ Server 2
      ↑                                    ↑
    User A                               User B

    User A sends to User B:
    Server1 → Redis channel → Server2 → User B
```

### Server-Sent Events (SSE)

```
  Client ────── HTTP Request ──────→ Server
  Client ←───── Event Stream ─────── Server
  Client ←───── Event ────────────── Server
  Client ←───── Event ────────────── Server

  One-directional (server → client)
  Auto-reconnection built-in

  Use Cases:
    - Notifications
    - Real-time feeds
    - Progress updates

  vs WebSocket:
    ✓ Simpler (just HTTP)
    ✓ Auto-reconnect
    ✓ Works through HTTP proxies
    ✗ No client → server streaming
    ✗ Text only (no binary)
```

### Long Polling vs Short Polling

```
Short Polling:
  Client: "Any updates?" → Server: "No"     (every 5 seconds)
  Client: "Any updates?" → Server: "No"
  Client: "Any updates?" → Server: "Yes! Here's data"

  ✗ Wasteful requests
  ✗ Not real-time (delay = poll interval)

Long Polling:
  Client: "Any updates?" → Server: [holds connection 30s]
  Server: "Here's data" (or timeout → reconnect)
  Client: "Any updates?" → Server: [holds connection]

  ✓ More real-time than short polling
  ✓ Works everywhere (simple HTTP)
  ✗ Server holds connections (resource intensive)
  ✗ Head-of-line blocking

Protocol Selection Guide:
  Need bidirectional?        → WebSocket
  Server → Client only?     → SSE
  Maximum compatibility?     → Long Polling
  Low-latency gaming/trading? → WebSocket or UDP
  Simple notifications?      → SSE
```

---

## 15.11 Microservices Architecture

### Monolith vs Microservices

```
Monolith:
  ┌────────────────────────────┐
  │  UI + Business Logic + DB  │
  │  All in one deployable     │
  └────────────────────────────┘

  ✓ Simple development and deployment
  ✓ Easy debugging (single process)
  ✓ No network overhead between components
  ✗ Single tech stack
  ✗ Hard to scale specific components
  ✗ One bug can bring down everything

Microservices:
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │ User │  │ Post │  │ Feed │  │ Chat │
  │ Svc  │  │ Svc  │  │ Svc  │  │ Svc  │
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │         │
  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐
  │Users│  │Posts│  │Feed │  │Chat │
  │ DB  │  │ DB  │  │Cache│  │ DB  │
  └─────┘  └─────┘  └─────┘  └─────┘

  ✓ Independent deployment
  ✓ Tech stack per service
  ✓ Independent scaling
  ✓ Team autonomy
  ✗ Network complexity
  ✗ Distributed system challenges
  ✗ Operational overhead
  ✗ Data consistency harder

Start Monolith → Extract Microservices as needed
(Don't start with microservices unless you have a large team)
```

### API Gateway

```
┌────────────────────────────────────────────┐
│                API Gateway                  │
│  ┌────────┐ ┌────────┐ ┌────────┐         │
│  │Auth    │ │Rate    │ │Routing │         │
│  │        │ │Limit   │ │        │         │
│  └────────┘ └────────┘ └────────┘         │
│  ┌────────┐ ┌────────┐ ┌────────┐         │
│  │Logging │ │Caching │ │Transform│        │
│  └────────┘ └────────┘ └────────┘         │
└──────┬───────────┬───────────┬─────────────┘
       │           │           │
   ┌───┴──┐   ┌───┴──┐   ┌───┴──┐
   │Svc A │   │Svc B │   │Svc C │
   └──────┘   └──────┘   └──────┘

Responsibilities:
  - Request routing
  - Authentication/Authorization
  - Rate limiting
  - Response caching
  - Request/Response transformation
  - SSL termination
  - Load balancing
  - Logging and monitoring

Tools: Kong, AWS API Gateway, Envoy, Nginx, Zuul
```

### Inter-Service Communication

```
Synchronous:
  REST:     Simple CRUD between services
  gRPC:     High-performance, streaming support

  ✓ Simple request/response
  ✗ Tight coupling, cascading failures
  ✗ Higher latency (chain of calls)

Asynchronous (Message-Based):
  Event Bus:    Services publish/subscribe to events
  Command Queue: One service sends command to another

  ✓ Loose coupling
  ✓ Resilient (message persisted in queue)
  ✓ Better scalability
  ✗ Complex debugging (distributed tracing needed)
  ✗ Eventual consistency

Service Mesh (Sidecar Pattern):
  ┌────────────────────┐
  │ Service A           │
  │  ┌──────┐ ┌──────┐ │
  │  │ App  │→│Proxy │ │  (Envoy sidecar)
  │  └──────┘ └──┬───┘ │
  └──────────────│─────┘
                 │ mTLS, load balancing,
                 │ retries, circuit breaking
  ┌──────────────│─────┐
  │ Service B    │      │
  │  ┌──────┐ ┌──┴───┐ │
  │  │ App  │←│Proxy │ │
  │  └──────┘ └──────┘ │
  └────────────────────┘

  Tools: Istio, Linkerd, Consul Connect

  ✓ Infrastructure concerns out of application code
  ✓ Consistent policies across services
  ✗ Operational complexity
  ✗ Latency overhead (extra hop)
```

---

## 15.12 Unique ID Generation

```python
import time
import threading

# 1. UUID v4 — Random, universally unique
import uuid
id_v4 = uuid.uuid4()  # e.g., "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
# ✓ Simple, no coordination
# ✗ 128 bits, not sortable, no time info

# 2. Snowflake ID (Twitter's approach)
class SnowflakeGenerator:
    """
    64-bit ID:
    [1 bit unused][41 bits timestamp][5 bits datacenter][5 bits machine][12 bits sequence]

    Can generate 4096 IDs per millisecond per machine
    Time-sortable! IDs are roughly chronologically ordered
    """
    EPOCH = 1288834974657  # Custom epoch (Twitter's)

    def __init__(self, datacenter_id, machine_id):
        assert 0 <= datacenter_id < 32
        assert 0 <= machine_id < 32
        self.datacenter_id = datacenter_id
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def generate(self):
        with self.lock:
            timestamp = int(time.time() * 1000) - self.EPOCH

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
                if self.sequence == 0:
                    # Wait for next millisecond
                    while timestamp <= self.last_timestamp:
                        timestamp = int(time.time() * 1000) - self.EPOCH
            else:
                self.sequence = 0

            self.last_timestamp = timestamp

            return ((timestamp << 22) |
                    (self.datacenter_id << 17) |
                    (self.machine_id << 12) |
                    self.sequence)

# 3. ULID — Lexicographically sortable, compatible with UUID
# 128 bits: [48 bits timestamp] [80 bits random]
# Example: 01ARZ3NDEKTSV4RRFFQ69G5FAV

# 4. Database Auto-Increment
# ✓ Simple, sequential
# ✗ Single point of failure, not distributed

# 5. Database Auto-Increment with Ranges
# Server 1: IDs 1, 3, 5, 7, ...   (step = N, offset = 1)
# Server 2: IDs 2, 4, 6, 8, ...   (step = N, offset = 2)
# ✗ Hard to add/remove servers

# Comparison:
# Method              │ Sortable │ Distributed │ Size    │ Collision
# ────────────────────│──────────│─────────────│─────────│──────────
# UUID v4             │ No       │ Yes         │ 128 bit │ Near-zero
# Snowflake           │ Yes      │ Yes         │ 64 bit  │ None (coordinated)
# ULID                │ Yes      │ Yes         │ 128 bit │ Near-zero
# Auto-increment      │ Yes      │ No          │ 64 bit  │ None
# DB range            │ Yes      │ Partial     │ 64 bit  │ None
```

---

## 15.13 DNS (Domain Name System)

```
DNS Resolution Flow:

  Browser ──→ Local DNS Cache ──→ ISP DNS Resolver
                                       │
                                       ↓
                              Root Name Server (.)
                                       │
                                       ↓
                              TLD Name Server (.com)
                                       │
                                       ↓
                              Authoritative Server (example.com)
                                       │
                                       ↓
                              IP Address: 93.184.216.34

Record Types:
  A      → Domain to IPv4 address
  AAAA   → Domain to IPv6 address
  CNAME  → Alias to another domain
  NS     → Name server for domain
  MX     → Mail server for domain
  TXT    → Text data (SPF, DKIM, verification)
  SRV    → Service location (port, priority, weight)

DNS for Load Balancing:
  - Round-robin DNS (multiple A records)
  - Weighted routing
  - Latency-based routing
  - Geolocation routing
  - Failover routing

TTL (Time to Live):
  Short TTL (60s): Faster failover, more DNS queries
  Long TTL (86400s): Fewer queries, slower failover
  Typical: 300s (5 minutes)
```

---

[← Previous: SD Fundamentals](14-system-design-fundamentals.md) | [Next: Distributed Systems →](16-distributed-systems.md)
