# Chapter 14: System Design Fundamentals

[← Previous: Specialized & Emerging](13-specialized-and-emerging.md) | [Next: System Design Building Blocks →](15-system-design-building-blocks.md)

---

# Part A: Core Concepts

## 14.1 What is System Design?

System design is the process of defining the architecture, components, modules, interfaces, and data flow of a system to satisfy specified requirements at scale.

```
System Design = Requirements + Architecture + Tradeoffs

Two types:
  1. High-Level Design (HLD): Architecture, components, data flow
  2. Low-Level Design (LLD): Classes, interfaces, data models, APIs
```

---

## 14.2 System Design Interview Framework

```
Step 1: Requirements Clarification (3-5 min)
  ├── Functional Requirements: What does the system do?
  ├── Non-Functional Requirements: Scale, latency, availability, consistency?
  └── Back-of-envelope estimation: QPS, storage, bandwidth

Step 2: High-Level Design (10-15 min)
  ├── Define API endpoints
  ├── Draw system architecture
  └── Identify core components

Step 3: Deep Dive (15-20 min)
  ├── Database schema & choice
  ├── Scaling strategies
  ├── Caching strategy
  └── Handle edge cases

Step 4: Wrap-Up (5 min)
  ├── Bottleneck identification
  ├── Monitoring & alerting
  └── Future improvements
```

---

## 14.3 Back-of-Envelope Estimation

### Power of 2 Reference

```
Power    Exact Value        Approx           Bytes
2^10     1,024              1 Thousand        1 KB
2^20     1,048,576          1 Million         1 MB
2^30     1,073,741,824      1 Billion         1 GB
2^40     1,099,511,627,776  1 Trillion        1 TB
2^50     ~1 Quadrillion                       1 PB
```

### Latency Numbers Every Programmer Should Know

```
Operation                              Time
─────────────────────────────────────────────────
L1 cache reference                     0.5 ns
Branch mispredict                      5 ns
L2 cache reference                     7 ns
Mutex lock/unlock                      100 ns
Main memory reference                  100 ns
Compress 1K bytes with Zippy           10 μs
Send 1 KB over 1 Gbps network         10 μs
Read 4 KB randomly from SSD            150 μs
Read 1 MB sequentially from memory     250 μs
Round trip within same datacenter      500 μs
Read 1 MB sequentially from SSD        1 ms
HDD seek                               10 ms
Read 1 MB sequentially from HDD        20 ms
Send packet CA → Netherlands → CA      150 ms

Key takeaways:
  - Memory is ~100x faster than SSD
  - SSD is ~10-100x faster than HDD
  - Network round-trip ≈ 0.5ms within DC, 150ms cross-continent
  - Avoid disk seeks; sequential >> random
  - Compress data before sending over network
```

### Common Estimation Templates

```python
# QPS (Queries Per Second) Estimation
daily_active_users = 100_000_000  # 100M DAU
queries_per_user_per_day = 10
total_queries_per_day = daily_active_users * queries_per_user_per_day
qps = total_queries_per_day / 86400  # ≈ 11,574 QPS
peak_qps = qps * 3  # ~35K QPS (3x average for peak)

# Storage Estimation
records_per_day = 50_000_000
record_size_bytes = 500
daily_storage = records_per_day * record_size_bytes  # 25 GB/day
yearly_storage = daily_storage * 365  # ~9 TB/year

# Bandwidth Estimation
incoming_data_per_second = qps * record_size_bytes  # bytes/sec
# Convert to Mbps: bytes/sec * 8 / 1,000,000

# Server Estimation
# A single modern server can handle:
# - 10K-50K concurrent connections (with async I/O)
# - 1K-10K QPS for typical web requests
# - 100K+ QPS for cached/simple lookups
```

### Quick Math Rules

```
1 day ≈ 100K seconds (86,400)
1 year ≈ 30M seconds (31,536,000)
1M requests/day ≈ 12 QPS
1B requests/day ≈ 12K QPS
1 char = 1 byte (ASCII) or 2-4 bytes (Unicode)
1 typical URL ≈ 100 bytes
1 tweet-sized text ≈ 500 bytes
1 image ≈ 300 KB (compressed)
1 video minute ≈ 50 MB (compressed)
```

---

## 14.4 Scalability

### Vertical Scaling (Scale Up)

```
Add more power to existing machine:
  ✓ Simple — no code changes
  ✓ No distributed complexity
  ✗ Hardware limits (can't add infinite RAM/CPU)
  ✗ Single point of failure
  ✗ Expensive at high end

Typical: Move from 8 cores → 64 cores, 32GB RAM → 512GB RAM
```

### Horizontal Scaling (Scale Out)

```
Add more machines:
  ✓ Theoretically unlimited
  ✓ Better fault tolerance
  ✓ Cost-effective (commodity hardware)
  ✗ Complex — distributed systems problems
  ✗ Need load balancing, data partitioning
  ✗ Network overhead

Typical: 1 server → 100 servers behind load balancer
```

### Scaling Dimensions

```
                    Read-Heavy              Write-Heavy
                    ─────────              ──────────
Caching             ★★★★★                  ★★☆☆☆
Read Replicas       ★★★★★                  ☆☆☆☆☆
Sharding            ★★★☆☆                  ★★★★★
CQRS                ★★★★☆                  ★★★★☆
Message Queues      ★★☆☆☆                  ★★★★★
CDN                 ★★★★★                  ☆☆☆☆☆
```

---

## 14.5 Availability & Reliability

### Availability Levels

```
Availability %    Downtime/Year    Downtime/Month    Called
──────────────    ─────────────    ──────────────    ──────
99%               3.65 days        7.31 hours        Two 9s
99.9%             8.77 hours       43.83 minutes     Three 9s
99.95%            4.38 hours       21.92 minutes     Three and a half 9s
99.99%            52.6 minutes     4.38 minutes      Four 9s
99.999%           5.26 minutes     26.3 seconds      Five 9s (gold standard)

Real examples:
  - AWS S3: 99.99% availability, 99.999999999% (11 nines) durability
  - Google Search: ~99.99%
  - Most web apps target 99.9% - 99.99%
```

### Availability in Series vs Parallel

```
                    Series (both must work):        Parallel (either works):
                    A(total) = A₁ × A₂              A(total) = 1 - (1-A₁)(1-A₂)

Example:
  Two components with 99.9% availability:
    Series:   0.999 × 0.999 = 99.8% (worse)
    Parallel: 1 - (0.001)² = 99.9999% (better)
```

### Redundancy Patterns

```
Active-Passive (Failover):
  ┌──────┐     ┌──────┐
  │Active│ ──→ │Passive│   Passive takes over when active fails
  └──────┘     └──────┘   + Simple, - Wasted resources, warm-up time

Active-Active:
  ┌──────┐     ┌──────┐
  │Active│ ←─→ │Active│   Both handle traffic, share load
  └──────┘     └──────┘   + No wasted resources, instant failover
                          - More complex, need conflict resolution

Multi-region:
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │US-East   │ ──→ │US-West   │ ──→ │EU-West  │
  └─────────┘     └─────────┘     └─────────┘
  + Disaster recovery, low latency globally
  - Expensive, data synchronization complexity
```

---

## 14.6 Consistency Models

### CAP Theorem

```
You can only have TWO out of three:

         Consistency
            /\
           /  \
          /    \
         / CAP  \
        / Theorem \
       /──────────\
  Availability ── Partition Tolerance

In distributed systems, network partitions WILL happen.
So the real choice is: CP or AP?

CP (Consistency + Partition Tolerance):
  - System returns error if data might be stale
  - Examples: HBase, MongoDB (default), Redis Cluster
  - Use: Banking, inventory systems

AP (Availability + Partition Tolerance):
  - System always responds, but data might be stale
  - Examples: Cassandra, DynamoDB, CouchDB
  - Use: Social media feeds, analytics
```

### PACELC Theorem (Extension of CAP)

```
If Partition → choose A or C
Else (normal) → choose Latency or Consistency

System          │ P: A or C │ E: L or C
────────────────│───────────│──────────
DynamoDB        │  PA       │  EL
Cassandra       │  PA       │  EL
MongoDB         │  PC       │  EC
MySQL (cluster) │  PC       │  EC
```

### Consistency Levels

```
Strong Consistency:
  - Every read returns the most recent write
  - Like a single machine
  - Examples: RDBMS, ZooKeeper, etcd
  - Cost: Higher latency, lower throughput

Eventual Consistency:
  - All replicas converge over time
  - Reads may return stale data temporarily
  - Examples: DNS, Cassandra, S3
  - Benefit: High availability, low latency

Causal Consistency:
  - Reads respect causal order (if A caused B, you see A before B)
  - Weaker than strong, stronger than eventual

Read-Your-Writes:
  - You always see your own writes
  - Others may see stale data
  - Common in user-facing applications

Linearizability:
  - Strongest: operations appear instantaneous at some point
  - Required by: consensus algorithms (Raft, Paxos)

Session Consistency:
  - Monotonic reads within a session
  - Good enough for most applications
```

---

## 14.7 Networking Fundamentals for System Design

### HTTP/HTTPS

```
HTTP Methods:
  GET     - Read (idempotent, cacheable)
  POST    - Create (not idempotent)
  PUT     - Update/Replace (idempotent)
  PATCH   - Partial update
  DELETE  - Remove (idempotent)

Status Codes:
  2xx Success:  200 OK, 201 Created, 204 No Content
  3xx Redirect: 301 Permanent, 302 Temporary, 304 Not Modified
  4xx Client:   400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many
  5xx Server:   500 Internal Error, 502 Bad Gateway, 503 Service Unavailable

HTTP/2 vs HTTP/1.1:
  - Multiplexing (multiple requests on one connection)
  - Header compression (HPACK)
  - Server push
  - Binary protocol (vs text)

HTTP/3 (QUIC):
  - Built on UDP (vs TCP)
  - 0-RTT connection establishment
  - Better for mobile (connection migration)
```

### Communication Protocols

```
Protocol       │ Direction    │ Latency │ Use Case
───────────────│──────────────│─────────│──────────────────
REST           │ Req/Resp     │ Med     │ CRUD APIs (most common)
GraphQL        │ Req/Resp     │ Med     │ Flexible queries, mobile
gRPC           │ Req/Resp     │ Low     │ Microservice-to-microservice
WebSocket      │ Bidirectional│ Very Low│ Chat, real-time
SSE            │ Server→Client│ Low     │ Notifications, feeds
Long Polling   │ Req/Resp     │ Med     │ Simple real-time fallback
UDP            │ Fire-forget  │ Lowest  │ Video streaming, gaming
```

### REST vs GraphQL vs gRPC

```
REST:
  + Simple, well-understood, cacheable
  + Stateless
  - Over-fetching / under-fetching
  - Multiple round trips for related data
  Best for: Public APIs, CRUD microservices

GraphQL:
  + Client specifies exact data needed
  + Single endpoint, one round trip
  + Strongly typed schema
  - Complex server implementation
  - Hard to cache
  - N+1 query problem
  Best for: Mobile apps, complex UIs, multiple data sources

gRPC:
  + Very fast (Protocol Buffers binary serialization)
  + Streaming support (unary, server, client, bidirectional)
  + Code generation (type-safe clients)
  - Not browser-friendly (needs proxy)
  - Not human-readable
  Best for: Internal microservices, low-latency inter-service
```

### API Design Best Practices

```
RESTful URL Design:
  GET    /api/v1/users              → List users
  POST   /api/v1/users              → Create user
  GET    /api/v1/users/{id}         → Get user
  PUT    /api/v1/users/{id}         → Update user
  DELETE /api/v1/users/{id}         → Delete user
  GET    /api/v1/users/{id}/posts   → Get user's posts

Pagination:
  Offset-based: ?page=3&limit=20     (simple, skip issues at scale)
  Cursor-based: ?cursor=abc123&limit=20 (for real-time feeds)
  Keyset:       ?after_id=500&limit=20  (efficient for sorted data)

Versioning:
  URL:    /api/v1/users    (most common)
  Header: Accept: application/vnd.api.v1+json
  Query:  /api/users?version=1

Rate Limiting Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 47
  X-RateLimit-Reset: 1625097600
  Retry-After: 3600
```

---

## 14.8 Proxies

### Forward Proxy

```
Client → [Forward Proxy] → Internet → Server

Use cases:
  - Hide client identity (anonymity)
  - Content filtering / access control
  - Caching for clients
  - Bypass restrictions

Examples: VPN, corporate proxy, Tor
```

### Reverse Proxy

```
Client → Internet → [Reverse Proxy] → Server(s)

Use cases:
  - Load balancing
  - SSL termination
  - Caching
  - Compression
  - DDoS protection
  - Rate limiting

Examples: Nginx, HAProxy, Cloudflare, AWS ALB
```

---

## 14.9 Consistent Hashing

Critical for distributed caching, databases, and partitioning.

```
Problem: Simple hash (key % N servers) breaks when adding/removing servers.
         Rehashing moves ~(N-1)/N keys = almost everything!

Consistent Hashing:
  - Map both servers and keys to a hash ring (0 to 2^32)
  - Key goes to nearest server clockwise
  - Adding/removing a server only affects ~K/N keys (minimal disruption)

     0 ────── Server A ──────
    /                          \
   |                            |
   |   Key1 → A                 |
   |   Key2 → B                 |
   |                            |
    \                          /
     ──── Server C ── Server B ──

Virtual Nodes:
  - Each physical server has multiple positions on ring
  - Improves balance (prevents hotspots)
  - Server A has A-1, A-2, A-3 on ring

Used by: Cassandra, DynamoDB, Memcached, Akka, Chord DHT
```

```python
import hashlib

class ConsistentHash:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:vn{i}"
            h = self._hash(virtual_key)
            self.ring[h] = node
            self.sorted_keys.append(h)
        self.sorted_keys.sort()

    def remove_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:vn{i}"
            h = self._hash(virtual_key)
            del self.ring[h]
            self.sorted_keys.remove(h)

    def get_node(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        # Find first server clockwise
        from bisect import bisect_right
        idx = bisect_right(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

# Usage
ch = ConsistentHash(["server1", "server2", "server3"])
print(ch.get_node("user:123"))  # Maps to a server
print(ch.get_node("user:456"))
# Adding server4 only remaps ~25% of keys (not 75%!)
ch.add_node("server4")
```

---

## 14.10 Data Partitioning (Sharding)

### Strategies

```
1. Horizontal Partitioning (Sharding):
   Split ROWS across databases

   Table: Users
   Shard 1: user_id 1-1M
   Shard 2: user_id 1M-2M
   Shard 3: user_id 2M-3M

2. Vertical Partitioning:
   Split COLUMNS/tables across databases

   DB1: User profiles (id, name, email)
   DB2: User posts (id, user_id, content)
   DB3: User media (id, user_id, url, size)

3. Directory-Based:
   Lookup service maps key → shard
   + Flexible, easy to change
   - Extra hop, SPOF if not replicated
```

### Sharding Keys (Partition Keys)

```
Requirements for a good partition key:
  ✓ Even distribution (no hotspots)
  ✓ Supports common query patterns
  ✓ Minimizes cross-shard queries

Examples:
  User service:  user_id (hash-based)
  Chat service:  chat_room_id (range-based keeps room together)
  Feed service:  user_id (all your feed on one shard)
  Geo service:   geographic region
  Time-series:   time bucket + entity_id

Common problems:
  - Celebrity/hotspot: One shard gets disproportionate traffic
    Fix: Further split hot keys, use caching
  - Cross-shard joins: Need data from multiple shards
    Fix: Denormalize, or use scatter-gather
  - Rebalancing: Adding shards requires data movement
    Fix: Consistent hashing, virtual shards
```

---

## 14.11 Replication

```
Single Leader (Master-Slave):
  ┌────────┐     ┌─────────┐
  │ Leader  │────→│Follower 1│
  │(writes) │────→│Follower 2│
  └────────┘     └─────────┘

  ✓ Simple
  ✓ Read scaling (read from followers)
  ✗ Write bottleneck (single leader)
  ✗ Failover complexity

  Sync vs Async replication:
    Sync: Leader waits for follower ACK → strong consistency, slower
    Async: Leader doesn't wait → eventual consistency, faster

Multi-Leader:
  ┌────────┐     ┌────────┐
  │Leader 1 │←──→│Leader 2 │
  └────────┘     └────────┘

  ✓ Write scaling
  ✓ Better latency (write to nearest leader)
  ✗ CONFLICT RESOLUTION needed
  ✗ Complex

  Conflict resolution:
    - Last-write-wins (LWW) — simple but lossy
    - Custom merge logic — application-specific
    - CRDTs — conflict-free replicated data types

Leaderless (Dynamo-style):
  Any node can serve reads/writes
  Uses quorum: W + R > N

    W=2, R=2, N=3: Strong consistency
    W=1, R=3, N=3: Fast writes
    W=3, R=1, N=3: Fast reads

  ✓ High availability
  ✓ No single point of failure
  ✗ Complex conflict resolution

  Used by: Cassandra, DynamoDB, Riak
```

---

## 14.12 Transactions & ACID

```
ACID Properties:
  Atomicity:    All or nothing (transaction completes fully or rolls back)
  Consistency:  Data satisfies all constraints after transaction
  Isolation:    Concurrent transactions don't interfere
  Durability:   Committed transactions survive crashes (in WAL/disk)

Isolation Levels (weakest → strongest):
  Level              │ Dirty Read │ Non-Repeatable │ Phantom Read
  ───────────────────│────────────│────────────────│─────────────
  Read Uncommitted   │ Yes        │ Yes            │ Yes
  Read Committed     │ No         │ Yes            │ Yes
  Repeatable Read    │ No         │ No             │ Yes
  Serializable       │ No         │ No             │ No

  Default for most RDBMS: Read Committed or Repeatable Read

Distributed Transactions:
  2PC (Two-Phase Commit):
    Phase 1: Coordinator asks all "Can you commit?" (prepare/vote)
    Phase 2: If all yes → "Commit!", else → "Abort!"

    Problem: Blocking if coordinator fails

  3PC: Non-blocking but more complex

  Saga Pattern (preferred in microservices):
    T1 → T2 → T3 → ... (each step is a local transaction)
    If T3 fails: C2 → C1 (compensating transactions in reverse)

    Choreography: Each service publishes events
    Orchestration: Central coordinator manages flow
```

### BASE (Alternative to ACID)

```
Basically Available:  System guarantees availability
Soft state:          State may change without input (due to eventual consistency)
Eventually consistent: System converges to consistent state

ACID                    BASE
─────                   ─────
Strong consistency      Eventual consistency
Pessimistic            Optimistic
Complex/slow at scale   Simpler/faster at scale
Traditional RDBMS      NoSQL databases
```

---

## 14.13 Security Fundamentals

### Authentication & Authorization

```
Authentication (Who are you?):
  - Password-based (bcrypt, argon2 hashing)
  - Token-based (JWT, OAuth 2.0)
  - Multi-factor (MFA/2FA)
  - Certificate-based (mTLS)
  - SSO (SAML, OpenID Connect)

Authorization (What can you do?):
  - RBAC (Role-Based Access Control)
  - ABAC (Attribute-Based Access Control)
  - ACL (Access Control List)
  - Policy-based (OPA / Cedar)

JWT (JSON Web Token):
  [Header].[Payload].[Signature]

  Header:  {"alg": "HS256", "typ": "JWT"}
  Payload: {"user_id": 123, "role": "admin", "exp": 1625097600}
  Signature: HMAC-SHA256(header + "." + payload, secret)

  ✓ Stateless (no server-side session)
  ✓ Can contain claims (role, permissions)
  ✗ Can't be revoked individually (use short expiry + refresh tokens)
  ✗ Size (800+ bytes vs session cookie)

OAuth 2.0 Flows:
  Authorization Code: Web apps (most secure)
  Client Credentials: Server-to-server
  Implicit: Single-page apps (deprecated)
  PKCE: Mobile/SPA (recommended replacement for implicit)
```

### Common Security Measures

```
Data in Transit:     TLS/SSL (HTTPS)
Data at Rest:        AES-256 encryption
API Security:        Rate limiting, API keys, OAuth
SQL Injection:       Parameterized queries, ORM
XSS:                 Input sanitization, CSP headers
CSRF:                CSRF tokens, SameSite cookies
DDoS:                CDN, rate limiting, WAF
Password Storage:    bcrypt/argon2 (never plain text!)
Secrets:             Vault (HashiCorp), AWS Secrets Manager
```

---

## 14.14 Monitoring & Observability

### Three Pillars

```
1. Metrics (Quantitative):
   - System: CPU, memory, disk, network
   - Application: QPS, latency (p50/p95/p99), error rate
   - Business: Revenue, signups, conversions
   Tools: Prometheus, Grafana, Datadog, CloudWatch

2. Logging (Events):
   - Structured logs (JSON format)
   - Log levels: DEBUG → INFO → WARN → ERROR → FATAL
   - Centralized logging (ELK stack, Splunk, Loki)
   - Correlation IDs for tracing across services

3. Tracing (Request Flow):
   - Distributed tracing across microservices
   - Trace ID propagated through all services
   - Identify bottlenecks and failures
   Tools: Jaeger, Zipkin, AWS X-Ray, OpenTelemetry
```

### Key Metrics

```
RED Method (for request-driven services):
  Rate:     Requests per second
  Errors:   Failed requests per second
  Duration: Distribution of request duration

USE Method (for resources):
  Utilization: % time resource is busy
  Saturation: Amount of work resource can't serve
  Errors: Count of error events

Four Golden Signals (Google SRE):
  1. Latency   — Time to serve a request
  2. Traffic   — Requests per second
  3. Errors    — Rate of failed requests
  4. Saturation — How "full" is the system
```

### SLI, SLO, SLA

```
SLI (Service Level Indicator):
  Metric that measures service quality
  Example: 95th percentile latency = 200ms

SLO (Service Level Objective):
  Target value for an SLI
  Example: p95 latency < 300ms for 99.9% of requests

SLA (Service Level Agreement):
  Contract with consequences for missing SLOs
  Example: If availability < 99.95%, customer gets credits

Error Budget:
  100% - SLO = budget for downtime/errors
  99.9% SLO → 0.1% error budget → 43 min/month downtime allowed
```

---

## 14.15 System Design Principles

### Key Principles

```
1. Keep It Simple (KISS)
   - Start simple, add complexity when needed
   - Premature optimization is the root of all evil

2. Separation of Concerns
   - Each component does one thing well
   - Microservices, layered architecture

3. Single Point of Failure (SPOF)
   - Identify and eliminate SPOFs
   - Redundancy at every layer

4. Stateless Design
   - Services don't store client state locally
   - State → external store (DB, cache, session store)
   - Enables horizontal scaling

5. Design for Failure
   - Everything fails: networks, disks, services
   - Implement retries, circuit breakers, fallbacks
   - Chaos engineering (Netflix Chaos Monkey)

6. Design for Scale
   - 10x your current load — is the architecture ready?
   - Identify bottlenecks before they hit you

7. Idempotency
   - Operations can be safely retried
   - Critical for payment systems, message processing
   - Use idempotency keys for POST requests
```

### Tradeoffs You'll Always Face

```
Tradeoff                Application
─────────               ───────────
Consistency vs Availability    CAP theorem
Latency vs Consistency         Sync replication vs async
Latency vs Throughput          Batching helps throughput, hurts latency
Complexity vs Flexibility      Microservices vs monolith
Read vs Write optimization     Denormalization helps reads, hurts writes
Cost vs Performance            More servers = faster but more $$
Accuracy vs Speed              Approximate algorithms, sampling
Security vs Usability          MFA, encryption add friction
```

---

[← Previous: Specialized & Emerging](13-specialized-and-emerging.md) | [Next: System Design Building Blocks →](15-system-design-building-blocks.md)
