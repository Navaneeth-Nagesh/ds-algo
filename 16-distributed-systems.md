# Chapter 16: Distributed Systems

[← Previous: SD Building Blocks](15-system-design-building-blocks.md) | [Next: Classic System Designs →](17-system-design-classic.md)

---

## 16.1 Consensus Algorithms

### Why Consensus?

```
Problem: Multiple nodes need to agree on a value/state even when:
  - Nodes may crash
  - Network may partition
  - Messages may be delayed or lost
  - NO clocks can be perfectly synchronized

Consensus is needed for:
  - Leader election
  - Distributed locks
  - Log replication
  - Configuration management
  - Atomic broadcast
```

### Paxos

```
Roles:
  Proposer:  Suggests a value
  Acceptor:  Votes on proposals
  Learner:   Learns the decided value

Phase 1 (Prepare):
  Proposer → Acceptor: "Prepare(n)" with proposal number n
  Acceptor → Proposer:
    If n > highest seen: "Promise(n, prev_accepted)"
    Else: Reject

Phase 2 (Accept):
  If majority promised:
    Proposer → Acceptor: "Accept(n, value)"
    Acceptor → Proposer:
      If no higher-numbered prepare received: "Accepted(n, value)"
      Else: Reject

Phase 3 (Learn):
  If majority accepted:
    Value is chosen! Notify learners.

Key Properties:
  Safety:   Only a single value is chosen
  Liveness: Progress guaranteed if majority is alive (eventually)

Problems:
  - Complex to implement correctly
  - Dueling proposers can livelock
  - Multiple rounds needed

Multi-Paxos: Single leader is pre-elected → skips Phase 1 for subsequent rounds
```

### Raft (Understandable Consensus)

```
Roles: Leader, Follower, Candidate

State Machine:
  Follower ──(timeout)──→ Candidate ──(wins election)──→ Leader
                              │                              │
                              └──(higher term seen)──→ Follower
                              └──(loses election)──→ Follower

Leader Election:
  1. Follower times out (no heartbeat from leader)
  2. Becomes Candidate, increments term, votes for self
  3. Requests votes from all other nodes
  4. Wins if gets majority (N/2 + 1) of votes
  5. Becomes Leader, starts sending heartbeats

Log Replication:
  1. Client sends command to Leader
  2. Leader appends to its log
  3. Leader replicates to Followers via AppendEntries RPC
  4. Once majority acknowledges → entry is committed
  5. Leader notifies Followers to apply (commit)
  6. Leader responds to client

  Leader:     [1:x←3] [2:y←1] [3:x←5] [4:z←2]
  Follower A: [1:x←3] [2:y←1] [3:x←5]          (slightly behind)
  Follower B: [1:x←3] [2:y←1] [3:x←5] [4:z←2]  (up to date)
  Follower C: [1:x←3] [2:y←1]                    (more behind)

  Committed up to index 3 (majority has it)

Safety Guarantees:
  - Election Safety: At most one leader per term
  - Leader Append-Only: Never overwrites/deletes entries
  - Log Matching: If two entries have same index+term, identical prefix
  - Leader Completeness: Committed entries in all future leaders' logs

Implementation Notes:
  - Term acts as logical clock
  - Randomized election timeout (150-300ms) prevents split votes
  - Heartbeat interval << election timeout

Used by: etcd, CockroachDB, TiDB, Consul, InfluxDB
```

```python
# Simplified Raft Node State
from enum import Enum
import random

class Role(Enum):
    FOLLOWER = 0
    CANDIDATE = 1
    LEADER = 2

class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers

        # Persistent state
        self.current_term = 0
        self.voted_for = None
        self.log = []  # list of (term, command)

        # Volatile state
        self.role = Role.FOLLOWER
        self.commit_index = 0
        self.last_applied = 0

        # Leader state
        self.next_index = {}   # peer → next log index to send
        self.match_index = {}  # peer → highest replicated index

    def start_election(self):
        self.current_term += 1
        self.role = Role.CANDIDATE
        self.voted_for = self.node_id
        votes_received = 1  # Vote for self

        # RequestVote RPC to all peers
        for peer in self.peers:
            vote = self.request_vote(peer, self.current_term,
                                     len(self.log), self.last_log_term())
            if vote:
                votes_received += 1

        majority = (len(self.peers) + 1) // 2 + 1
        if votes_received >= majority:
            self.become_leader()
        else:
            self.role = Role.FOLLOWER

    def become_leader(self):
        self.role = Role.LEADER
        for peer in self.peers:
            self.next_index[peer] = len(self.log)
            self.match_index[peer] = 0
        # Start sending heartbeats

    def last_log_term(self):
        return self.log[-1][0] if self.log else 0

    def request_vote(self, peer, term, last_log_index, last_log_term):
        # Simplified — in reality this is an RPC
        pass

    def append_entries(self, peer, entries):
        # Simplified — Leader replicates log entries
        pass
```

---

## 16.2 Distributed Clocks & Ordering

### The Clock Problem

```
Physical clocks drift:
  Node A: 12:00:00.000
  Node B: 12:00:00.157  (157ms ahead)
  Node C: 11:59:59.843  (157ms behind)

  NTP can synchronize to ~10ms, but not perfect
  Google TrueTime: ~7ms uncertainty (atomic clocks + GPS)

Ordering events across nodes is HARD without perfect clocks.
```

### Lamport Timestamps (Logical Clocks)

```
Rules:
  1. Before each event: L(e) = counter++
  2. Before sending message: attach L(e)
  3. On receive: counter = max(counter, received_L) + 1

  Node A:  [1] ──send──→ [2]           [6]
                          │
  Node B:        [1] [2]  │  [3] ──send──→
                           ↓              │
  Node C:               [3] [4]  [5]  ←──┘ [6]

Property:
  If a → b (a happened before b), then L(a) < L(b)
  BUT: L(a) < L(b) does NOT mean a → b
  (Can't distinguish causality from concurrency)
```

### Vector Clocks

```
Each node maintains a vector of counters [N₁, N₂, N₃]

Rules:
  1. Before event on node i: V[i]++
  2. Sending: attach full vector
  3. On receive: V[i] = max(V[i], received_V[i]) for all i, then V[self]++

  Node A:  [1,0,0] → [2,0,0] ───send───→ [3,0,0]
                                    │
  Node B:  [0,1,0]              ←───┘  [2,2,0] → [2,3,0]
                                                      │
  Node C:  [0,0,1] → [0,0,2]                    ←────┘ [2,3,3]

Comparing:
  V1 = V2:         V1[i] = V2[i] for all i → SAME event
  V1 < V2:         V1[i] ≤ V2[i] for all i, and at least one strict <
                    → V1 happened before V2
  V1 || V2:        Neither < nor = → CONCURRENT (conflict!)

Used by: DynamoDB, Riak for conflict detection
Problem: Vector size = number of nodes (doesn't scale well)
Fix: Dotted version vectors, interval tree clocks
```

### Hybrid Logical Clocks (HLC)

```
Combines physical clock + logical counter:
  HLC = (physical_time, logical_counter, node_id)

Better than pure logical: respects real-time ordering
Better than pure physical: handles clock skew

Used by: CockroachDB, MongoDB
```

---

## 16.3 Gossip Protocol

```
How Gossip Works (Epidemic Protocol):

Round 1: Node A has update
  A → randomly tells B and C

Round 2: A, B, C each tell 2 random peers
  Now 3 → 6-9 nodes know

Round 3: Those tell 2 random peers each
  Now most nodes know

Convergence: O(log N) rounds to reach all N nodes

Types:
  Anti-Entropy: Periodically sync full state with random peer
  Rumor Mongering: Spread new updates to random peers

Properties:
  ✓ Scalable (O(log N) convergence)
  ✓ Fault-tolerant (no single point of failure)
  ✓ Decentralized
  ✗ Eventually consistent (not instant)
  ✗ Network bandwidth overhead
  ✗ Redundant messages (same info sent multiple times)

Used by:
  - Cassandra (membership, failure detection)
  - Consul (cluster membership)
  - SWIM protocol (membership)
  - Bitcoin/blockchain networks
```

### Failure Detection

```
Heartbeat-Based:
  Each node periodically sends "I'm alive" to others
  If no heartbeat in timeout → suspicion → marked dead
  Problems: Can't distinguish slow from dead

Phi Accrual Detector (Cassandra):
  Instead of binary alive/dead:
  - Compute suspicion level φ (phi) based on heartbeat history
  - φ = -log10(P(heartbeat_is_late))
  - Higher φ → more suspicious
  - Threshold (e.g., φ > 8) → mark as dead

  ✓ Adaptive to network conditions
  ✓ Fewer false positives

SWIM Protocol:
  1. Pick random node → send "ping"
  2. If no ACK → pick k random nodes → "ping-req(target)"
  3. If none get ACK → mark target as suspect
  4. After timeout → mark as failed

  ✓ O(1) messages per node per round
  ✓ Subgroup-based (fast, reliable)
```

---

## 16.4 Distributed File Systems

### Google File System (GFS) / HDFS

```
Architecture:
  ┌──────────────────┐
  │  Master / NameNode │  (metadata, chunk locations, namespace)
  └────────┬─────────┘
           │
  ┌────────┼────────┐
  ┌──┴───┐ ┌──┴───┐ ┌──┴───┐
  │Chunk  │ │Chunk  │ │Chunk  │
  │Server1│ │Server2│ │Server3│   (DataNodes - store actual data)
  └──────┘ └──────┘ └──────┘

Key Design:
  - Files split into large chunks (64MB-256MB)
  - Each chunk replicated 3x across different racks
  - Master is single point, but has hot standby
  - Optimized for large sequential reads/writes
  - Append-only (no random writes)

Write Flow (GFS):
  1. Client asks Master for chunk locations
  2. Client sends data to nearest chunk server
  3. Data propagated chain-wise: CS1 → CS2 → CS3
  4. Client sends write request to primary chunk server
  5. Primary orders and applies writes
  6. Primary forwards to secondaries
  7. All acknowledge → Client notified
```

---

## 16.5 MapReduce

```
Two Phases:
  Map:    (key, value) → list of (intermediate_key, intermediate_value)
  Reduce: (intermediate_key, list of values) → (key, final_value)

Word Count Example:
  Input:  "the cat sat on the mat"

  Map Phase (parallel):
    Mapper 1: "the cat sat" → [(the,1), (cat,1), (sat,1)]
    Mapper 2: "on the mat"  → [(on,1), (the,1), (mat,1)]

  Shuffle & Sort (framework handles):
    cat: [1]
    mat: [1]
    on:  [1]
    sat: [1]
    the: [1, 1]

  Reduce Phase (parallel):
    Reducer: (cat, [1]) → (cat, 1)
    Reducer: (the, [1,1]) → (the, 2)
    ...

  Result: {cat:1, mat:1, on:1, sat:1, the:2}

MapReduce Architecture:
  ┌───────┐
  │ Job   │
  │Tracker│
  └───┬───┘
      │ assigns
  ┌───┴───┐───────┐───────┐
  │Mapper 1│Mapper 2│Mapper 3│  → local disk (intermediate)
  └───┬───┘───┬───┘───┬───┘
      └───────┼───────┘
              ↓ shuffle
  ┌──────────┐──────────┐
  │ Reducer 1 │ Reducer 2 │  → output (HDFS)
  └──────────┘──────────┘

Limitations:
  - Disk I/O between stages (slow)
  - Not great for iterative algorithms
  → Solution: Apache Spark (in-memory, DAG of transformations)
```

---

## 16.6 Distributed Locking

```
Why Distributed Locks?
  - Prevent concurrent modifications to shared resources
  - Leader election
  - Task deduplication
  - Rate limiting

Redis-based Lock (Redlock algorithm):
  1. Get current time
  2. Try to acquire lock on N/2+1 Redis instances
  3. Calculate elapsed time
  4. If lock acquired on majority AND elapsed < lock TTL → success
  5. Otherwise → release lock on all instances

  SET resource_name my_random_value NX PX 30000
  (NX = only if not exists, PX = expire in 30s)

  Release: Check value first (prevent deleting someone else's lock)
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    end

ZooKeeper-based Lock:
  1. Create ephemeral sequential node /locks/resource/lock-0000001
  2. If your node has lowest sequence → you have the lock
  3. Otherwise, watch the node just before yours
  4. When it's deleted → check again

  ✓ Fair ordering (FIFO)
  ✓ Ephemeral nodes auto-cleanup on disconnect

Fencing Tokens:
  Problem: Lock holder pauses (GC), lock expires, another acquires
  Solution: Include monotonically increasing fence token with lock
  Resource checks: "Is this token > last seen?" before accepting writes

  ┌───────┐  lock(token=33) ┌────────┐
  │Client A│───────────────→│Resource│
  └───────┘                 └────────┘
  [GC pause...]              ... token 34 acquired by B ...
  ┌───────┐  write(token=33)┌────────┐
  │Client A│───────────────→│Resource│ → REJECTED (33 < 34)
  └───────┘                 └────────┘
```

---

## 16.7 Distributed Transactions

### Two-Phase Commit (2PC)

```
Coordinator (Transaction Manager):

Phase 1 - PREPARE:
  Coordinator → Participant A: "Can you commit?"
  Coordinator → Participant B: "Can you commit?"
  A → Coordinator: "Yes" (or "No")
  B → Coordinator: "Yes"

Phase 2 - COMMIT/ABORT:
  If all "Yes":
    Coordinator → A: "COMMIT"
    Coordinator → B: "COMMIT"
  If any "No":
    Coordinator → A: "ABORT"
    Coordinator → B: "ABORT"

Problems:
  - Blocking: If coordinator crashes after Phase 1, participants stuck
  - Single point of failure: Coordinator
  - Performance: Locks held during both phases
```

### Saga Pattern

```
For long-lived transactions across microservices:

Forward: T1 → T2 → T3 → T4 (each is a local transaction)
If T3 fails: C2 → C1 (compensating transactions in reverse)

Example: Book a Trip
  T1: Reserve flight         C1: Cancel flight reservation
  T2: Reserve hotel          C2: Cancel hotel reservation
  T3: Charge credit card     C3: Refund credit card
  T4: Send confirmation      C4: Send cancellation email

Choreography (Event-based):
  Each service publishes events, next service reacts
  ┌──────┐ FlightReserved ┌──────┐ HotelReserved ┌──────┐
  │Flight│───────────────→│Hotel │───────────────→│Payment│
  └──────┘                └──────┘                └──────┘

  ✓ Simple, loose coupling
  ✗ Hard to understand flow, cyclic dependencies

Orchestration (Central coordinator):
  ┌──────────────┐
  │  Saga        │
  │  Orchestrator│
  └──────┬───────┘
    ┌────┼────┐
  ┌─┴──┐ ┌┴──┐ ┌┴────┐
  │Flight│ │Hotel│ │Payment│
  └─────┘ └────┘ └──────┘

  ✓ Easy to understand, centralized logic
  ✗ Single point of failure, can become complex
```

---

## 16.8 Consistent Hashing (Advanced)

### Virtual Nodes & Rebalancing

```
Without Virtual Nodes:
  Ring: [A]────────────[B]────[C]
  Problem: Uneven distribution when nodes have different capacity

With Virtual Nodes:
  Ring: [A1]──[B2]──[C1]──[A2]──[B1]──[C2]──[A3]──[B3]──[C3]

  Node A (powerful):  3 virtual nodes
  Node B (medium):    3 virtual nodes
  Node C (small):     3 virtual nodes

  Benefits:
    - Even distribution regardless of number of physical nodes
    - When node fails, load spreads evenly (not all to one neighbor)
    - Easy rebalancing: change virtual node count

Replication with Consistent Hashing:
  Key → walk clockwise → replicate to next N distinct physical nodes

  Preference List: [Primary, Replica1, Replica2]
  Skip virtual nodes of same physical node
```

---

## 16.9 Bloom Filters in Distributed Systems

```
Probabilistic membership test:
  - "Is X in set?" → "Definitely not" or "Probably yes"
  - False positives possible, false negatives NEVER
  - Space efficient: ~10 bits per element for 1% FP rate

Distributed Use Cases:
  - Cassandra: Avoid unnecessary disk reads for missing keys
  - CDN edge: Quick check if content is cached
  - Distributed cache: Avoid network round-trip for missing keys
  - Web crawler: Detect already-visited URLs
  - Spam filter: Quick check against blacklist

Counting Bloom Filter:
  - Supports deletions (counters instead of bits)
  - More space, but allows remove()

Cuckoo Filter:
  - Supports deletions
  - Better space efficiency than counting BF
  - Faster lookups
```

---

## 16.10 Data Replication Strategies

### Synchronous vs Asynchronous

```
Synchronous Replication:
  Client → Primary → [write] → Replica (wait for ACK) → Client ACK

  ✓ Strong consistency (no data loss)
  ✗ Higher latency (must wait for replica)
  ✗ Lower availability (if replica is slow/down)

Asynchronous Replication:
  Client → Primary → [write] → Client ACK
  Primary → Replica (background, eventually)

  ✓ Low latency
  ✓ High availability
  ✗ Potential data loss if primary fails
  ✗ Stale reads from replica

Semi-Synchronous:
  Primary → (at least 1 replica ACK) → Client ACK
  Other replicas replicate async

  Compromise: Some durability guarantee without full sync cost
```

### Conflict Resolution

```
Last Writer Wins (LWW):
  - Timestamp determines winner
  - Simple but lossy (concurrent writes dropped)
  - Used by: Cassandra

Application-Level Resolution:
  - Return all conflicting versions to client
  - Client decides how to merge
  - Used by: DynamoDB (via conditional writes)

CRDTs (Conflict-free Replicated Data Types):
  Data structures that automatically resolve conflicts:

  G-Counter: Grow-only counter
    Node A: {A:3, B:0, C:0}
    Node B: {A:0, B:5, C:0}
    Merge:  {A:3, B:5, C:0} → total = 8

  PN-Counter: Positive-Negative counter (supports decrement)
  G-Set: Grow-only set (add only)
  OR-Set: Observed-Remove set (add and remove)
  LWW-Register: Last-write-wins register

  ✓ Always converge, no conflicts
  ✓ No coordination needed
  ✗ Limited data structure types
  ✗ Storage overhead

  Used by: Redis (CRDTs for active-active), Riak, Automerge
```

### Change Data Capture (CDC)

```
Capture database changes as a stream of events:

  Database → [CDC] → [Kafka] → Search Index
                             → Cache Invalidation
                             → Analytics
                             → Another Database

Methods:
  Log-based: Read database WAL/binlog (least intrusive)
  Trigger-based: DB triggers write to change table
  Query-based: Periodically poll for changes

Tools: Debezium, Maxwell, DynamoDB Streams, MongoDB Change Streams

Use Cases:
  - Keep Elasticsearch in sync with primary DB
  - Invalidate cache when data changes
  - Event sourcing from existing DB
  - Cross-datacenter replication
```

---

## 16.11 Distributed Caching

### Cache Coherence Strategies

```
Write-Invalidate:
  On write: Invalidate cached copies
  Next read: Cache miss → fetch fresh data
  ✓ Simple, saves bandwidth
  ✗ Cache miss after every write

Write-Update:
  On write: Update all cached copies
  ✓ No cache misses after write
  ✗ Network bandwidth for unused updates

Lease-Based:
  Cache entry has a lease (TTL)
  During lease: Cache serves reads without checking source
  Lease expires: Must revalidate
  ✓ Reduced load on source
  ✗ Stale during lease period
```

### Memcached vs Redis

```
Feature          │ Memcached          │ Redis
─────────────────│────────────────────│──────────────────
Data structures  │ String only        │ String, Hash, List, Set, ZSet, Stream
Persistence      │ No                 │ RDB snapshots + AOF
Replication      │ No (client-side)   │ Master-Replica
Clustering       │ Client-side        │ Built-in (Redis Cluster)
Threading        │ Multi-threaded     │ Single-threaded (6.0+ I/O threads)
Memory           │ Slab allocator     │ jemalloc
Max value size   │ 1 MB               │ 512 MB
Pub/Sub          │ No                 │ Yes
Lua scripting    │ No                 │ Yes
Transactions     │ No                 │ MULTI/EXEC

When to use Memcached:
  - Simple key-value caching
  - Multi-threaded needed
  - Memory efficiency for simple strings

When to use Redis:
  - Need data structures (sorted sets, lists)
  - Need persistence
  - Need pub/sub
  - Almost always (Redis is more versatile)
```

---

## 16.12 Leader Election

```
Bully Algorithm:
  - Node with highest ID becomes leader
  - On suspecting leader failure:
    1. Send ELECTION to all higher-ID nodes
    2. If no response → you're the leader, broadcast VICTORY
    3. If response → wait for someone higher to win

ZooKeeper Election:
  - Each node creates ephemeral sequential znode
  - Node with lowest sequence number is leader
  - Others watch the node just before them
  - On leader failure: ephemeral node disappears, next takes over

Raft Election:
  - Random timeout triggers candidacy
  - Candidate requests votes from all
  - Majority vote wins
  - Term number prevents stale elections
  (See Section 16.1 for full details)

etcd / Consul:
  - Built on Raft consensus
  - Provides distributed lock + leader election out of the box
  - Most practical choice for production systems
```

---

## 16.13 Partition Strategies

### Range Partitioning

```
Partition by key ranges:
  Shard 1: A-H
  Shard 2: I-P
  Shard 3: Q-Z

✓ Efficient range queries (all data in one shard)
✓ Sequential scans within range
✗ Hotspots (popular letters get more traffic)
✗ Uneven distribution

Used by: HBase (region splits on size), CockroachDB
```

### Hash Partitioning

```
Partition by hash of key:
  Shard = hash(key) % num_shards

✓ Even distribution
✓ No hotspots (if hash is good)
✗ Range queries impossible (related keys scattered)
✗ Resharding requires data movement

Used by: Cassandra, DynamoDB, Redis Cluster
```

### Composite/Hybrid Partitioning

```
Combine hash + range:
  First level: hash(user_id) → shard
  Within shard: range(timestamp) → partition

Example (Cassandra):
  PRIMARY KEY ((user_id), timestamp)
  Partition key: user_id (hash-distributed)
  Clustering key: timestamp (sorted within partition)

  ✓ Even distribution across shards
  ✓ Efficient range queries within a partition
```

---

## 16.14 Idempotency

```
Definition: Performing an operation multiple times gives the same result as once.

Why it matters:
  - Network retries may duplicate requests
  - Message queues may deliver twice (at-least-once)
  - Client may re-submit on timeout

Idempotency Key Pattern:
  1. Client generates unique idempotency key
  2. Server checks if key was already processed
  3. If yes → return cached result
  4. If no → process, store result with key

  POST /api/payments
  Idempotency-Key: abc-123-def
  {amount: 100, to: "user456"}

  Server:
    if key "abc-123-def" in processed_requests:
        return cached_response      # Don't charge twice!
    else:
        process_payment()
        save_response(key, response)
        return response

Implementation Options:
  - Database unique constraint on idempotency key
  - Redis SET NX with TTL
  - Deduplication table

Naturally Idempotent Operations:
  GET:    Always idempotent
  PUT:    Idempotent (same state regardless of repeats)
  DELETE: Idempotent (deleting twice = same as once)
  POST:   NOT idempotent (need idempotency key)
```

---

## 16.15 Backpressure & Flow Control

```
Problem: Producer is faster than consumer → system overwhelmed

Strategies:
  1. Drop: Discard excess messages (acceptable for metrics, logs)
  2. Buffer: Queue messages (bounded buffer → overflow issue)
  3. Sample: Process every Nth message (analytics)
  4. Backpressure: Signal producer to slow down

Backpressure Mechanisms:
  TCP flow control:     Receiver advertises window size
  Reactive Streams:     Subscriber requests N items at a time
  Message Queue:        Consumer ACKs before getting next batch
  Circuit Breaker:      Reject requests when overloaded
  Rate Limiting:        Hard cap on incoming rate
  Load Shedding:        Drop low-priority requests under load

  Priority-Based Shedding:
    Normal load:  Process everything
    High load:    Drop analytics, process transactions
    Critical:     Drop non-essential, process only critical path
```

---

## 16.16 Geo-Distribution & Multi-Region

```
Active-Passive (Disaster Recovery):
  ┌─────────┐ replication  ┌─────────┐
  │ US-East  │────────────→│ EU-West  │
  │ (Active) │             │ (Standby)│
  └─────────┘             └─────────┘

  + Simple
  - DR site doesn't serve traffic (wasted)
  - Failover time (minutes to hours)

Active-Active (Multi-Master):
  ┌─────────┐ ←——————→ ┌─────────┐
  │ US-East  │          │ EU-West  │
  │ (Active) │          │ (Active) │
  └─────────┘          └─────────┘

  Users routed to nearest region (GeoDNS)

  + Low latency everywhere
  + Zero downtime failover
  - Conflict resolution needed
  - Data consistency challenges

Data Residency:
  Some data MUST stay in specific regions (GDPR, etc.)
  - User data stays in user's region
  - Reference data replicated everywhere
  - Cross-region queries through API (not direct DB access)

Global Load Balancing:
  GeoDNS → nearest healthy region
  Anycast → same IP, nearest edge
  AWS Global Accelerator, Cloudflare
```

---

## 16.17 Consistency Patterns in Practice

### Read-Your-Own-Writes

```
Problem: User writes to leader, reads from follower, doesn't see own write

Solutions:
  1. Read from leader for own data (within 30s of write)
  2. Client remembers write timestamp → wait for replica to catch up
  3. Sticky sessions: Same user → same replica
```

### Monotonic Reads

```
Problem: User reads from Replica A (fresh), then Replica B (stale)
         Appears as if time went backward

Solution:
  - Same user always reads from same replica (hash user_id → replica)
  - Or: track read timestamp, reject stale replicas
```

### Quorum Reads/Writes

```
N = total replicas
W = write quorum (must succeed)
R = read quorum (must read from)

Strong Consistency: W + R > N

Example (N=3):
  W=2, R=2: Strong (always overlap)
  W=1, R=3: Fast writes, slow reads, strong
  W=3, R=1: Slow writes, fast reads, strong
  W=1, R=1: Fast both, eventual consistency

Sloppy Quorum (Dynamo):
  If primary nodes unavailable, write to any available node
  Later: "hinted handoff" moves data to correct node
  Trade: Availability over consistency
```

---

[← Previous: SD Building Blocks](15-system-design-building-blocks.md) | [Next: Classic System Designs →](17-system-design-classic.md)
