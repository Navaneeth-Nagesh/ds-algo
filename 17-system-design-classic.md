# Chapter 17: Classic System Designs

[← Previous: Distributed Systems](16-distributed-systems.md) | [Next: Design Patterns →](18-design-patterns.md)

---

## 17.1 URL Shortener (TinyURL / Bit.ly)

### Requirements

```
Functional:
  - Given a long URL, return a short URL
  - Redirect short URL to original
  - Optional: custom alias, expiration, analytics

Non-Functional:
  - 100M URLs generated/day, 10:1 read:write ratio → 1B redirects/day
  - Low latency redirects (<50ms)
  - High availability
  - Short URLs 7-8 chars
```

### Design

```
API:
  POST /api/v1/shorten   { longUrl, customAlias?, expireDate? } → { shortUrl }
  GET  /{shortCode}       → 301/302 Redirect to longUrl

Architecture:
  Client → [Load Balancer] → [API Servers] → [Cache (Redis)] → [Database]
                                  ↓
                          [ID Generator]

Short URL Generation:
  Option 1: Base62 encoding of auto-increment ID
    ID: 11157 → base62 → "dnh"
    Characters: [a-z, A-Z, 0-9] = 62 chars
    7 chars → 62^7 = 3.5 trillion combinations

  Option 2: MD5/SHA256 hash → take first 7 chars
    Collision handling: check DB, append counter

  Option 3: Pre-generated key service (KGS)
    Generate keys in advance → store unused in DB
    On request: pop a key from unused pool
    ✓ No collision, O(1)
    ✗ KGS is a dependency

Database Schema:
  urls:
    id            BIGINT PRIMARY KEY
    short_code    VARCHAR(8) UNIQUE INDEX
    long_url      VARCHAR(2048)
    user_id       BIGINT INDEX
    created_at    TIMESTAMP
    expires_at    TIMESTAMP
    click_count   BIGINT DEFAULT 0

Redirect Flow:
  1. User hits short URL
  2. Check Redis cache → if hit, redirect (most cases)
  3. Cache miss → query DB → cache result → redirect
  4. Return 301 (permanent, browser caches) or 302 (temporary, track clicks)

Scaling:
  - Cache: 20% of URLs handle 80% of traffic → cache hot URLs
  - DB: Range-based sharding on short_code first char
  - Analytics: Async write to Kafka → analytics DB
```

---

## 17.2 Rate Limiter

### Requirements

```
Functional:
  - Limit requests per user/IP/API key per time window
  - Return 429 Too Many Requests when exceeded
  - Headers: X-RateLimit-Remaining, Retry-After

Non-Functional:
  - Very low latency (on every request)
  - Distributed (multiple API servers share limit)
  - Fault tolerant (if rate limiter down, allow traffic)
```

### Design

```
Architecture:
  Client → [API Gateway / Rate Limiter] → [API Servers]
                     ↓
              [Redis Cluster]
              (counters per user per window)

Placement:
  Option 1: Client-side (easy to bypass)
  Option 2: Server-side middleware
  Option 3: API Gateway (Kong, AWS API GW) ← most common

Algorithm Choice:
  - Token Bucket: Most flexible, allows bursts (used by AWS, Stripe)
  - Sliding Window: Most accurate

Redis Implementation:
  Key: "rate_limit:{user_id}:{window}"
  Value: counter

  MULTI
    INCR rate_limit:user123:1625097600
    EXPIRE rate_limit:user123:1625097600 60
  EXEC

Rules Configuration:
  rules:
    - endpoint: "/api/v1/messages"
      limits:
        - period: 1s
          max: 5
        - period: 1min
          max: 100
        - period: 1day
          max: 10000

Edge Cases:
  - Rate limiter down → fail open (allow all traffic)
  - Race conditions → Redis Lua script for atomicity
  - Distributed → eventual consistency OK (brief over-allowance)
```

---

## 17.3 Notification System

### Requirements

```
Functional:
  - Send push notifications (iOS, Android), SMS, email
  - Support templates, scheduling, prioritization
  - User notification preferences

Scale: 10M+ notifications/day
```

### Design

```
Architecture:
  ┌──────────┐     ┌──────────────┐     ┌───────────┐
  │ Services  │────→│ Notification │────→│   Kafka    │
  │ (trigger) │     │    Service   │     │  (buffer)  │
  └──────────┘     └──────────────┘     └─────┬─────┘
                                              │
                          ┌───────────────────┼────────────────┐
                          ↓                   ↓                ↓
                   ┌──────────┐      ┌──────────┐     ┌──────────┐
                   │Push Worker│      │SMS Worker │     │Email     │
                   │(APNS/FCM)│      │(Twilio)   │     │Worker    │
                   └──────────┘      └──────────┘     └──────────┘

Components:
  1. Notification Service: Validates, templates, checks preferences
  2. Message Queue (Kafka): Buffers, retry, decouple
  3. Workers: Type-specific delivery (push, SMS, email)
  4. User Preference DB: Opt-in/out per channel
  5. Template Service: Render templates with variables
  6. Rate Limiter: Don't spam users
  7. Analytics: Track delivery, open rates

Reliability:
  - Exactly-once delivery: Idempotency key per notification
  - Retry with exponential backoff
  - Dead letter queue for failed notifications
  - Delivery status tracking (sent, delivered, read)

Priority Levels:
  P0 (Critical): OTP, security alerts → bypass rate limits
  P1 (High):     Payment confirmations → fast delivery
  P2 (Normal):   Social notifications → can be batched
  P3 (Low):      Marketing → batch, respect quiet hours
```

---

## 17.4 Chat System (WhatsApp / Slack)

### Requirements

```
Functional:
  - 1:1 messaging with real-time delivery
  - Group chat (up to 500 members)
  - Online/offline status
  - Read receipts
  - Media sharing
  - Message history

Scale: 50M DAU, 1B messages/day
```

### Design

```
Architecture:
  ┌────────┐                     ┌────────┐
  │Client A│←──WebSocket──→┌─────┤Client B│
  └────────┘               │     └────────┘
                           │
  ┌──────────────┐    ┌────┴────────────┐
  │ Connection   │←──→│  Chat Service    │
  │ Manager      │    │ (routing logic)  │
  │ (WebSocket   │    └────┬────────────┘
  │  servers)    │         │
  └──────────────┘    ┌────┴────────────┐
                      │  Message Queue   │
                      │  (Kafka)         │
                      └────┬────────────┘
                      ┌────┴────────────┐
                      │    Database      │
                      │ (messages store) │
                      └─────────────────┘

Message Flow (User A → User B):
  1. A sends message via WebSocket to Chat Server
  2. Chat Server looks up B's connection (Connection Manager)
  3. If B is online → route message directly via WebSocket
  4. If B is offline → store in DB, send push notification
  5. When B comes online → pull unread messages from DB
  6. Message stored in DB for history regardless

Database Design:
  Messages (Cassandra — wide-column for time-series):
    PRIMARY KEY ((conversation_id), message_id)
    Partition key: conversation_id
    Clustering key: message_id (time-sorted, Snowflake ID)

    conversation_id │ message_id │ sender_id │ content    │ timestamp
    ────────────────│────────────│───────────│────────────│──────────
    conv_123        │ msg_001    │ user_A    │ "Hey!"     │ 12:00:01
    conv_123        │ msg_002    │ user_B    │ "Hi there" │ 12:00:05

  User Status (Redis):
    Key: "online:{user_id}" → last_seen_timestamp
    TTL: 300s (renew on heartbeat)

Group Chat:
  Message sent once → fanned out to each member's inbox
  Fan-out-on-write: Store copy per user (read fast, write heavy)
  Fan-out-on-read:  Store once, each user reads from group (write fast)

  Hybrid: Fan-out-on-write for small groups (<500)
          Fan-out-on-read for broadcast channels

End-to-End Encryption (Signal Protocol):
  - Key exchange: X3DH (Extended Triple Diffie-Hellman)
  - Message encryption: Double Ratchet algorithm
  - Server never sees plaintext
  - Forward secrecy: Compromised key doesn't decrypt old messages
```

---

## 17.5 Social Media Feed (Twitter / Instagram)

### Requirements

```
Functional:
  - Post content (text, images, video)
  - Follow/unfollow users
  - News feed (timeline of followed users' posts)
  - Like, comment, share

Scale: 300M MAU, 500M tweets/day, timeline reads 10x writes
```

### Design

```
Feed Generation — Two Approaches:

Fan-out on Write (Push Model):
  When User A posts:
    For each follower of A:
      Insert post into follower's timeline cache

  ┌───────┐   post   ┌──────────┐   push    ┌──────────────┐
  │User A  │────────→│ Post Svc  │─────────→│ Timeline Cache│
  └───────┘         └──────────┘   (Redis)  │ per user      │
  (1000 followers → 1000 writes)            └──────────────┘

  ✓ Fast reads (timeline pre-computed)
  ✗ Slow writes for celebrities (millions of followers)
  ✗ Wasted work for inactive users

Fan-out on Read (Pull Model):
  When User B opens timeline:
    Get list of users B follows
    Fetch recent posts from each
    Merge and sort by time

  ✓ No write amplification
  ✗ Slow reads (N queries to fetch, merge, sort)

Hybrid (Twitter's Approach):
  Normal users: Fan-out on write (push to followers' timelines)
  Celebrities (>100K followers): Fan-out on read

  When rendering timeline:
    1. Get pre-computed timeline from cache (followed non-celebrities)
    2. Fetch celebrity posts separately
    3. Merge and return

Architecture:
  ┌──────────────┐     ┌──────────────┐     ┌───────────────┐
  │ Post Service  │────→│ Fan-out Svc  │────→│Timeline Cache │
  │ (write posts) │     │ (async, Kafka)│    │(Redis per user)│
  └──────────────┘     └──────────────┘     └───────────────┘

  ┌──────────────┐     ┌──────────────┐
  │Timeline Service│←──│Timeline Cache │
  │ (read timeline)│   │+ Celebrity   │
  └──────────────┘    │  fetch        │
                      └──────────────┘

Database:
  Posts (sharded by user_id):
    post_id, user_id, content, media_urls, created_at

  Follows (graph):
    follower_id, followee_id, created_at
    Index on both (who I follow, who follows me)

  Timeline Cache (Redis sorted set per user):
    ZADD timeline:{user_id} {timestamp} {post_id}
    ZREVRANGE timeline:{user_id} 0 49  (latest 50 posts)
```

---

## 17.6 Video Streaming (YouTube / Netflix)

### Requirements

```
Functional:
  - Upload videos
  - Stream videos (adaptive bitrate)
  - Search, recommend
  - Like, comment, subscribe

Scale: 1B DAU, 500 hours of video uploaded per minute
```

### Design

```
Upload Pipeline:
  ┌────────┐     ┌──────────┐     ┌───────────────┐
  │ Client  │────→│Upload Svc │────→│  Object Store  │ (raw video)
  └────────┘     └──────────┘     └───────┬───────┘
                                          │
                                  ┌───────┴───────┐
                                  │ Transcoding    │
                                  │ Pipeline       │
                                  │ (parallel)     │
                                  └───────┬───────┘
                                          │
                          ┌───────────────┼───────────────┐
                          ↓               ↓               ↓
                    ┌──────────┐   ┌──────────┐   ┌──────────┐
                    │ 240p     │   │ 720p     │   │ 1080p    │
                    └──────────┘   └──────────┘   └──────────┘
                          ↓               ↓               ↓
                    ┌─────────────────────────────────────────┐
                    │              CDN (edge nodes)            │
                    └─────────────────────────────────────────┘

Video Transcoding:
  - Convert to multiple resolutions (240p, 360p, 480p, 720p, 1080p, 4K)
  - Multiple codecs (H.264, H.265/HEVC, VP9, AV1)
  - DAG (Directed Acyclic Graph) of tasks:
    Video → split into segments → transcode each parallel → merge
    Audio → extract → transcode → multiple formats
    Thumbnail → extract → multiple sizes

Adaptive Bitrate Streaming:
  HLS (HTTP Live Streaming) / DASH:
    1. Video split into small segments (2-10 seconds)
    2. Each segment available in multiple qualities
    3. Client monitors bandwidth
    4. Switches quality per segment based on network

    Manifest file (.m3u8):
      #EXT-X-STREAM-INF:BANDWIDTH=800000
      low/segment001.ts
      #EXT-X-STREAM-INF:BANDWIDTH=2400000
      mid/segment001.ts
      #EXT-X-STREAM-INF:BANDWIDTH=6000000
      high/segment001.ts

Recommendation System:
  Collaborative Filtering: Users who watched X also watched Y
  Content-Based: Similar category, tags, description
  Hybrid: Combine both + deep learning embeddings

  Pipeline: User features + Video features → ML model → Ranked list
  Two stages:
    Candidate Generation: Narrow from millions to thousands
    Ranking: Score and sort top candidates
```

---

## 17.7 Ride Sharing (Uber / Lyft)

### Requirements

```
Functional:
  - Rider requests a ride
  - Match rider with nearby driver
  - Real-time tracking
  - Pricing (surge pricing)
  - Payment processing

Scale: 20M rides/day, 5M drivers, real-time location updates
```

### Design

```
Architecture:
  ┌──────────┐         ┌──────────────┐         ┌──────────────┐
  │Rider App │←WebSocket→│ Trip Service  │←WebSocket→│ Driver App  │
  └──────────┘         └──────┬───────┘         └──────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ↓                    ↓                    ↓
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │Location Svc  │   │Matching Svc  │   │Pricing Svc   │
  │(driver pos)  │   │(assign driver)│   │(fare calc)   │
  └──────────────┘   └──────────────┘   └──────────────┘

Location Tracking:
  Drivers send GPS coordinates every 3-5 seconds

  Storage: GeoHash-based index

  GeoHash: Encode lat/lon into string
    (37.7749, -122.4194) → "9q8yyz"
    Prefix match = nearby: "9q8yy*" finds all drivers in area

  Implementation:
    Redis GEO commands:
      GEOADD drivers {lon} {lat} "driver_123"
      GEORADIUS drivers {lon} {lat} 5 km COUNT 20

    Or: QuadTree / S2 Geometry (Google's approach)
      - Divide world into cells of varying size
      - Dense areas = smaller cells
      - Efficient spatial queries

Matching Algorithm:
  1. Rider requests ride with pickup location
  2. Query nearby available drivers (radius search)
  3. Score candidates: distance + ETA + driver rating + acceptance rate
  4. Send request to top driver
  5. If declined/timeout → next driver
  6. If accepted → trip created

  ETA Calculation:
    Not straight-line distance!
    Use road network graph + real-time traffic
    Precomputed routing tables (Contraction Hierarchies)

Surge Pricing:
  demand_supply_ratio = active_requests / available_drivers in area
  surge_multiplier = f(demand_supply_ratio)

  ┌────────────────────────────────────┐
  │ Ratio    │ Multiplier              │
  │ < 1.0    │ 1.0x (normal)           │
  │ 1.0-1.5  │ 1.0-1.5x               │
  │ 1.5-2.0  │ 1.5-2.5x               │
  │ > 2.0    │ 2.5-5.0x               │
  └────────────────────────────────────┘

  Geospatial surge: city divided into hexagonal cells
  Each cell has independent surge calculation
```

---

## 17.8 Distributed Search Engine (Google)

### Architecture

```
Web Crawler → Indexer → Search Server → Ranking → Results

Crawling:
  ┌──────────┐     ┌───────────┐     ┌────────────┐
  │URL Frontier│───→│ Crawler    │───→│ Content    │
  │(queue)     │    │ (parallel) │    │ Store      │
  └──────────┘     └───────────┘     └────────────┘

  URL Frontier:
    - Priority queue (important pages first)
    - Politeness: Don't hammer one domain
    - robots.txt compliance
    - Bloom filter to avoid re-crawling
    - Freshness: Re-crawl based on change frequency

Indexing:
  1. Parse HTML, extract text
  2. Tokenize, stem, remove stop words
  3. Build inverted index: term → [doc_id, position, frequency]
  4. Store forward index: doc_id → [title, URL, snippet, PageRank]

  Distributed Index:
    - Sharded by term (term partitioning) or by document (doc partitioning)
    - Term partitioning: Each shard has all docs for some terms
    - Doc partitioning: Each shard has all terms for some docs
    - In practice: doc partitioning + scatter-gather queries

Ranking (Simplified):
  Score = Relevance × Quality × Freshness × Personalization

  PageRank:
    Each page has a score based on incoming links
    PR(A) = (1-d) + d × Σ(PR(T)/C(T))
    d = damping factor (~0.85)
    T = pages linking to A
    C(T) = outbound links from T

  Modern: BERT/Transformer models for relevance

Query Processing:
  1. Parse and understand query (NLP, spell check, synonyms)
  2. Scatter query to all index shards
  3. Each shard returns top-K results
  4. Merge, re-rank, diversify results
  5. Return top 10 with snippets
```

---

## 17.9 Distributed File Storage (Dropbox / Google Drive)

### Requirements

```
Functional:
  - Upload/download/sync files
  - Share files and folders
  - Automatic sync across devices
  - Version history, conflict resolution

Scale: 500M users, 15+ PB storage, high sync reliability
```

### Design

```
Architecture:
  ┌──────────────┐     ┌──────────────┐     ┌───────────────┐
  │Desktop Client │────→│ API Servers   │────→│ Metadata DB   │
  │              │     │              │     │(PostgreSQL)   │
  └──────────────┘     └──────────────┘     └───────────────┘
                              │
                       ┌──────┴──────┐
                       │Sync Service  │
                       │(notifications│
                       │ + conflict)  │
                       └──────┬──────┘
                              │
                       ┌──────┴──────┐
                       │Block Storage │
                       │(S3 / custom) │
                       └─────────────┘

Chunking (Key Optimization):
  File → split into 4MB chunks → each chunk gets content hash

  Benefits:
    1. Delta sync: Only upload changed chunks (not entire file)
    2. Deduplication: Same content → same hash → store once
    3. Parallel upload/download of chunks
    4. Resume interrupted transfers

  Example:
    file.pdf (100MB) → 25 chunks
    Edit page 3 → only 1-2 chunks change → upload 4-8 MB not 100 MB!

Metadata DB:
  Files:    file_id, name, parent_folder, owner_id, latest_version
  Chunks:   chunk_id, chunk_hash, file_id, sequence, size
  Versions: version_id, file_id, chunks_list, modified_at, modified_by
  Shares:   share_id, resource_id, user_id, permission

Sync Protocol:
  1. Client monitors local file system (inotify/FSEvents)
  2. File change detected → compute new chunk hashes
  3. Upload only new/changed chunks
  4. Update metadata (new version, ref chunks)
  5. Notify other devices via long polling/WebSocket
  6. Other devices download changed chunks, reassemble

Conflict Resolution:
  File edited on two offline devices:
    1. First sync wins → becomes current version
    2. Second sync detects conflict → creates "conflicted copy"
    3. User manually merges

  Google Docs approach: OT (Operational Transform) or CRDTs for real-time
```

---

## 17.10 Web Crawler

### Design

```
Architecture:
  ┌───────────┐     ┌───────────┐     ┌─────────────┐
  │URL Frontier│───→│  Fetcher   │───→│   Parser     │
  │(priority Q)│    │(HTTP client│    │(extract links│
  └─────┬─────┘    │ DNS cache) │    │ + content)   │
        ↑          └───────────┘    └──────┬──────┘
        │                                   │
        │          ┌───────────┐            │
        └──────────│URL Filter  │←──────────┘
                   │(dedup, robot│
                   │ politeness)│
                   └───────────┘

URL Frontier Design:
  Front Queues (Priority):
    Queue 1 (High):   Important domains (.edu, .gov, news sites)
    Queue 2 (Medium): Regular pages
    Queue 3 (Low):    Deep pages, less important

    Prioritizer assigns URLs to queues based on:
      PageRank, domain authority, freshness, relevance

  Back Queues (Politeness):
    One queue per domain
    Ensure minimum delay between requests to same domain

    Domain: example.com → Queue → [url1, url2, url3]
    Timer: Don't dequeue until 1s after last request to domain

Deduplication:
  - URL dedup: Bloom filter (billions of URLs, ~few GB memory)
  - Content dedup: SimHash / MinHash (detect near-duplicate pages)

Handling:
  - DNS caching (avoid repeated DNS lookups)
  - Respect robots.txt and rate limits
  - Handle redirects (limit depth)
  - Trap detection (infinite calendars, parametrized URLs)
  - Dynamic content: Headless browser (Puppeteer/Playwright)

Scale:
  - 1B pages to crawl
  - Distributed across hundreds of crawler machines
  - Each machine: multi-threaded, async I/O
  - ~1000 pages/sec per machine → 1000 machines → 1M pages/sec
```

---

## 17.11 Typeahead / Autocomplete

### Design

```
Requirements:
  - Return top 5-10 suggestions as user types
  - < 100ms latency
  - Based on popularity/frequency

Data Structure: Trie with top-K at each node

  root
  ├── t
  │   ├── tr
  │   │   ├── tre [tree:50, trend:45, treat:30]
  │   │   │   ├── tree [tree:50]
  │   │   │   ├── tren [trend:45]
  │   │   │   └── trea [treat:30]
  │   ├── tw [twitter:100, twitch:80]
  │   │   ├── twi [twitter:100, twitch:80]

Optimization:
  - Store top-K results at each trie node (precomputed)
  - No need to traverse further — return cached top-K
  - Trie fits in memory (limited vocabulary)

Architecture:
  ┌────────┐     ┌──────────┐     ┌──────────────┐
  │Client  │────→│ API Server│────→│Trie Service   │
  │(debounce│    └──────────┘     │(in-memory)    │
  │ 200ms) │                      └──────────────┘
  └────────┘                             ↑ rebuild
                                   ┌─────┴────────┐
                                   │Analytics (Spark│
                                   │aggregate freq)│
                                   └──────────────┘

Data Collection:
  1. Log all search queries with timestamp
  2. Periodic job (hourly/daily): aggregate frequencies
  3. Rebuild trie from aggregated data
  4. Trend detection: weight recent queries higher

Client Optimization:
  - Debounce: Don't query on every keystroke (wait 200ms)
  - Cache: Browser caches prefix → results
  - Pre-fetch: After "app", pre-fetch "appl" results
```

---

## 17.12 Payment System

### Requirements

```
Functional:
  - Process payments (credit card, bank transfer, wallet)
  - Handle refunds
  - Transaction history

Non-Functional:
  - EXACTLY-ONCE processing (no double charges!)
  - High reliability (99.999%)
  - PCI DSS compliance
  - Audit trail
```

### Design

```
Architecture:
  ┌──────────┐     ┌──────────────┐     ┌───────────────┐
  │ Client    │────→│Payment Service│────→│Payment Gateway│
  │           │     │              │     │(Stripe/Adyen) │
  └──────────┘     └──────┬───────┘     └───────────────┘
                          │
                   ┌──────┴───────┐
                   │  Ledger DB    │
                   │(double-entry) │
                   └──────────────┘

Double-Entry Bookkeeping:
  Every transaction creates TWO entries:
    Debit:  Buyer's account  -$100
    Credit: Seller's account +$100

  Sum of all debits = Sum of all credits (always balanced)

Payment Flow:
  1. Client sends payment request with idempotency key
  2. Payment service validates (amount, fraud check)
  3. Reserve funds (authorization)
  4. Call payment gateway (Stripe, PayPal)
  5. On success: capture payment, update ledger
  6. On failure: release authorization
  7. Send receipt/notification

Idempotency (Critical):
  POST /api/payments
  Idempotency-Key: "order_12345_payment_1"

  Server checks:
    1. Key exists in idempotency table? → return cached result
    2. New key → process payment → store result with key

  Prevents: Double-charging on retry, network timeout, client retry

Reconciliation:
  Daily batch job:
    1. Compare internal ledger with payment gateway records
    2. Compare with bank statements
    3. Flag discrepancies for manual review

  Types of mismatch:
    - Missing: Internal has record, gateway doesn't (or vice versa)
    - Amount: Different amounts recorded
    - Status: Different completion status
```

---

## 17.13 Distributed Key-Value Store (DynamoDB)

### Design

```
Architecture (Dynamo-style):
  - Consistent hashing for partitioning
  - Leaderless replication (quorum R/W)
  - Vector clocks for conflict detection
  - Gossip protocol for membership
  - Merkle trees for anti-entropy

Write Path:
  1. Client writes to any node (coordinator)
  2. Coordinator determines partition via consistent hash
  3. Write forwarded to N replica nodes
  4. Wait for W acknowledgments → success

Read Path:
  1. Client reads from coordinator
  2. Coordinator reads from R replica nodes
  3. Return most recent version (compare vector clocks)
  4. If stale replica found → read repair (async update)

Merkle Trees (Anti-Entropy):
  Compare data between replicas efficiently:

       H(1-4)                   H(1-4)
      /      \                 /      \
   H(1-2)   H(3-4)        H(1-2)   H(3-4)  ← different!
   /    \    /    \        /    \    /    \
  H1   H2  H3   H4      H1   H2  H3'  H4   ← H3 differs

  Only sync the subtree that differs (H3 in this case)
  O(log n) comparison to find differing keys

Tunable Consistency:
  N=3, W=2, R=2: Strong consistency
  N=3, W=1, R=1: Eventually consistent, fast
  Per-request tuning possible
```

---

## 17.14 Ticket Booking System (BookMyShow)

### Requirements

```
Functional:
  - Browse events, search, select seats
  - Reserve seats temporarily (hold for 10 min)
  - Process payment
  - Generate tickets

Challenges:
  - Double booking prevention
  - Hot events: thousands competing for same seats
  - Fairness: first-come-first-served
```

### Design

```
Seat Reservation (Critical Section):

  Option 1: Pessimistic Locking (SELECT FOR UPDATE)
    BEGIN;
    SELECT * FROM seats WHERE event_id=1 AND seat_id=42 FOR UPDATE;
    -- Check if available
    UPDATE seats SET status='HELD', held_by=user_1, held_until=NOW()+10min;
    COMMIT;

    ✓ Simple, correct
    ✗ Contention under high load (lock per seat)

  Option 2: Optimistic Locking (Version Check)
    SELECT version FROM seats WHERE seat_id=42;  -- version=5
    UPDATE seats SET status='HELD', version=6
    WHERE seat_id=42 AND version=5;
    -- If affected_rows=0 → conflict, retry

    ✓ Better for read-heavy
    ✗ Retries under contention

  Option 3: Redis + Lua (High Performance)
    Use Redis SETNX for atomic seat holds
    SET seat:event1:42 user_1 NX EX 600  -- hold for 600 seconds

    ✓ Very fast
    ✗ Need sync back to DB for durability

Architecture:
  ┌────────┐    ┌───────────┐    ┌────────────────┐
  │Browser │───→│API Gateway │───→│Booking Service  │
  └────────┘    └───────────┘    └───────┬────────┘
                                         │
           ┌─────────────────────────────┼──────────────┐
           ↓                             ↓              ↓
  ┌──────────────┐            ┌──────────────┐  ┌──────────────┐
  │Seat Inventory│            │Payment Svc   │  │Ticket Svc    │
  │(Redis + DB)  │            │              │  │(PDF + email) │
  └──────────────┘            └──────────────┘  └──────────────┘

Queue for Hot Events:
  Virtual waiting room:
    1. Users enter queue (random position or FIFO)
    2. Batch N users every 30 seconds into booking page
    3. Each batch has 10 minutes to select and pay
    4. Prevents thundering herd
```

---

## 17.15 Distributed Task Scheduler (Cron)

### Design

```
Requirements:
  - Schedule one-time and recurring tasks
  - Exactly-once execution guarantee
  - Handle worker failures
  - Scale to millions of scheduled tasks

Architecture:
  ┌──────────────┐    ┌────────────────┐    ┌───────────────┐
  │ Task Manager  │───→│ Task Queue     │───→│  Workers       │
  │ (API + store) │   │ (priority +    │    │  (execute)     │
  └──────────────┘    │  time-based)   │    └───────────────┘
                      └────────────────┘

Task Storage:
  - Database for task definitions (schedule, handler, params)
  - Sorted set in Redis: ZADD tasks {execution_time} {task_id}

  Scheduler loop:
    while True:
      now = time.time()
      tasks = ZRANGEBYSCORE tasks 0 now LIMIT 100
      for task in tasks:
        if acquire_lock(task):  # Prevent double execution
          enqueue_to_workers(task)
          ZREM tasks task
          if task.is_recurring:
            next_run = compute_next(task.cron_expr)
            ZADD tasks next_run task.id

Exactly-Once:
  1. Worker picks task from queue
  2. Worker acquires distributed lock (Redis SETNX + TTL)
  3. Execute task
  4. Mark complete in DB + release lock
  5. If worker dies: lock expires → task retried by another worker
  6. Task handler must be idempotent (retries safe)

Scaling:
  - Partition tasks by time range (shard by day/hour)
  - Multiple scheduler instances (only one active via leader election)
  - Workers scale independently
  - Priority lanes: separate queues for high/low priority
```

---

## 17.16 Design Google Maps / Navigation

### Requirements

```
Functional: Display map tiles, point-to-point navigation, ETA estimation,
  POI search, real-time traffic, turn-by-turn directions
Non-Functional: 1B users, fast route computation (<1s), global coverage
```

### Architecture

```
┌──────────────┐     ┌─────────────┐     ┌─────────────────┐
│ Mobile / Web │────→│  API Gateway │────→│  Routing Engine  │
│   Client     │     └──────┬──────┘     │  (A* / CH)       │
└──────────────┘            │            └────────┬─────────┘
       │                    │                     │
┌──────▼──────┐     ┌──────▼──────┐     ┌────────▼────────┐
│ Tile Server  │     │  POI Search  │     │ Traffic Service  │
│ (vector/raster)│   │ (Elasticsearch)│   │ (real-time data) │
└──────────────┘     └──────────────┘     └──────────────────┘
       │                                          │
┌──────▼──────┐                          ┌────────▼────────┐
│ CDN (tiles)  │                          │ GPS Data Pipeline│
│              │                          │ (Kafka → Flink)  │
└──────────────┘                          └──────────────────┘
```

### Key Decisions

```
1. Map Storage: Divide globe into tiles at multiple zoom levels (Z0=1 tile → Z18=2^36 tiles)
   Use quad-tree tiling: each tile subdivides into 4

2. Routing Engine:
   - Preprocess: Contraction Hierarchies (CH) — reduce graph by removing
     unimportant nodes. Preprocessing: hours; Query: microseconds
   - Runtime: Bidirectional A* on contracted graph
   - Real-time traffic: overlay live speed data on edges

3. ETA Estimation:
   - Historical speed data per road segment per time-of-day
   - ML model trained on GPS traces (actual travel times)
   - Blends historical + real-time traffic

4. Tile Serving: Pre-rendered raster tiles OR vector tiles (smaller, styled client-side)
   CDN caching with geographic distribution

5. POI Search: Geospatial index (R-tree / GeoHash) + inverted text index
   Proximity-weighted ranking: score = relevance × f(distance)

6. Live Traffic: Millions of GPS pings → Kafka → stream processor →
   compute segment speeds → update routing graph edge weights
```

---

## 17.17 Design E-Commerce Platform (Amazon)

### Requirements

```
Functional: Product catalog, search, cart, checkout, order tracking,
  reviews/ratings, recommendations, seller management
Non-Functional: 500M products, 100K orders/sec during peak, 99.99% checkout availability
```

### Architecture

```
┌──────────┐     ┌─────────┐     ┌──────────────────────────────────────────┐
│ Client   │────→│ API GW / │────→│         Microservices                    │
│ (Web/App)│     │ CDN      │     │ ┌─────────┐ ┌────────┐ ┌─────────────┐ │
└──────────┘     └──────────┘     │ │ Product  │ │ Search │ │   Cart      │ │
                                  │ │ Catalog  │ │ (ES)   │ │ (Redis)     │ │
                                  │ └────┬─────┘ └───┬────┘ └──────┬──────┘ │
                                  │ ┌────▼─────┐ ┌───▼────┐ ┌──────▼──────┐ │
                                  │ │Inventory │ │ Order  │ │  Payment    │ │
                                  │ │ Service  │ │ Service│ │  Service    │ │
                                  │ └──────────┘ └───┬────┘ └─────────────┘ │
                                  │              ┌───▼────────────────────┐  │
                                  │              │ Notification / Shipping│  │
                                  │              └────────────────────────┘  │
                                  └──────────────────────────────────────────┘
                                  ┌────────────────────────────────────────┐
                                  │ Recommendation Engine (collaborative   │
                                  │ filtering + content-based + real-time) │
                                  └────────────────────────────────────────┘
```

### Key Decisions

```
1. Product Catalog: Sharded by category/seller, denormalized for reads
   Document DB (DynamoDB) for flexible schemas per category

2. Search: Elasticsearch cluster, tokenized + faceted search
   Ranking: TF-IDF + sales velocity + reviews + personalization

3. Cart: Redis (fast reads/writes, TTL for guest carts)
   Merge guest cart with user cart on login

4. Inventory: Write-heavy; eventual consistency OK for display,
   STRONG consistency for checkout (reserve → confirm → deduct)
   Distributed lock per SKU during checkout

5. Order Pipeline (Saga pattern):
   Reserve Inventory → Process Payment → Confirm Order → Notify → Ship
   Each step has compensating action (rollback)

6. Recommendations:
   - Collaborative filtering: "users who bought X also bought Y"
   - Content-based: item attribute similarity
   - Real-time: session-based CTR prediction model

7. Flash Sales / High Traffic:
   - Virtual waiting queue (SQS/Redis)
   - Pre-warm inventory counts in cache
   - Rate limit per user
```

---

## 17.18 Design Collaborative Editing (Google Docs)

### Requirements

```
Functional: Real-time co-editing, cursor/selection awareness, comments,
  version history, offline editing, permissions
Non-Functional: <100ms edit propagation, handle 100 concurrent editors/doc
```

### Architecture

```
┌──────────┐   WebSocket    ┌──────────────────┐
│ Editor   │◄──────────────►│ Collaboration    │
│ Client   │                │ Server (session) │
└──────────┘                └───────┬──────────┘
                                    │
                            ┌───────▼──────────┐
                            │ Operation Log    │
                            │ (append-only)    │
                            └───────┬──────────┘
                                    │
                   ┌────────────────┼────────────────┐
                   │                │                 │
           ┌───────▼──────┐ ┌──────▼──────┐ ┌───────▼─────┐
           │ Document     │ │ Presence    │ │ Version     │
           │ Store        │ │ Service     │ │ History     │
           │ (snapshot)   │ │ (cursors)   │ │ (snapshots) │
           └──────────────┘ └─────────────┘ └─────────────┘
```

### Key Decisions

```
1. Conflict Resolution — Two approaches:

   OT (Operational Transformation):      CRDT (Conflict-Free Replicated Data Types):
   ┌─────────────────────────────┐        ┌───────────────────────────────┐
   │ Transform concurrent ops    │        │ Data structure guarantees     │
   │ against each other          │        │ convergence                   │
   │ Server is source of truth   │        │ No central server needed      │
   │ Used by: Google Docs        │        │ Used by: Figma, Yjs           │
   │ Simpler for text editing    │        │ Better for P2P / offline      │
   └─────────────────────────────┘        └───────────────────────────────┘

2. Operation Log: Append-only log of all operations per document
   Periodically create snapshots (every N ops) for fast loading

3. Presence Service: Broadcast cursor positions via WebSocket
   Heartbeat for active/idle detection, ephemeral — no persistence needed

4. Offline Support: Queue operations locally, sync on reconnect
   OT/CRDT resolves conflicts on merge

5. Version History: Store snapshots at checkpoints
   Reconstruct any version by replaying ops from nearest snapshot

6. Scaling: Shard by document ID, each doc pinned to one server (sticky session)
   One collaboration server handles all users editing the same doc
```

---

## 17.19 Design Social Network (LinkedIn / Facebook)

### Requirements

```
Functional: User profiles, connections/friends, news feed, messaging,
  people-you-may-know, search, groups, notifications
Non-Functional: 1B users, 500M DAU, O(1) connection check, real-time notifications
```

### Architecture

```
┌─────────┐     ┌──────────┐     ┌──────────────────────────────────┐
│ Client  │────→│ API GW   │────→│ Services                         │
└─────────┘     └──────────┘     │ ┌──────────┐  ┌───────────────┐  │
                                 │ │ Profile   │  │ Graph Service  │  │
                                 │ │ Service   │  │ (Neo4j / TAO) │  │
                                 │ └──────────┘  └───────┬───────┘  │
                                 │ ┌──────────┐  ┌───────▼───────┐  │
                                 │ │ Feed     │  │ Recommendation │  │
                                 │ │ Service  │  │ Engine (PYMK)  │  │
                                 │ └──────────┘  └───────────────┘  │
                                 └──────────────────────────────────┘
```

### Key Decisions

```
1. Social Graph Storage:
   - Adjacency list in KV store: user_id → [friend_ids] (fast O(1) friend check)
   - Graph DB (Neo4j/TAO) for traversals (2nd/3rd degree connections)
   - Facebook TAO: distributed graph cache over MySQL

2. People You May Know (PYMK):
   - Friend-of-friend with score = mutual_connections / total_connections
   - Exclude existing connections, blocked users
   - Precomputed offline (Spark job), refreshed daily
   - Online: fetch top-K from precomputed list, re-rank with real-time signals

3. News Feed: Hybrid fan-out (same as Twitter design, Ch 17.5)
   - Ranking model: affinity × edge_weight × time_decay
   - ML features: post type, author engagement history, user interests

4. Search:
   - People search: prefix trie + second-degree boost
   - Content search: Elasticsearch with social graph proximity weighting

5. Connection degrees (1st/2nd/3rd):
   - BFS from viewer up to depth 3 is expensive
   - Precompute 2nd-degree bloom filter per user
   - At query time: check 1st (O(1) set), check 2nd (bloom filter), else 3rd+

6. Notifications: Fan-out on write for connection updates
   Priority queue per user, real-time via WebSocket/SSE
```

---

## 17.20 Design Stock Exchange / Order Matching

### Requirements

```
Functional: Submit/cancel orders, order book, real-time price feed,
  trade execution, portfolio tracking
Non-Functional: <1ms matching latency, 1M orders/sec, exactly-once execution
```

### Architecture

```
┌──────────┐     ┌──────────────┐     ┌───────────────────────┐
│ Trader   │────→│  Gateway     │────→│  Matching Engine      │
│ Client   │     │  (validation,│     │  (in-memory order book)│
└──────────┘     │   rate limit)│     └──────────┬────────────┘
                 └──────────────┘                │
                                        ┌────────▼────────┐
                                        │ Trade Execution  │
                                        │ (settlement)     │
                                        └────────┬────────┘
                                                 │
                 ┌──────────────┐       ┌────────▼────────┐
                 │ Market Data  │◄──────│ Event Store     │
                 │ Feed (pub/sub│       │ (order log)     │
                 │ WebSocket)   │       └─────────────────┘
                 └──────────────┘
```

### Key Decisions

```
1. Order Book: Price-time priority (FIFO within same price)
   - Buy side: max-heap by price, FIFO within price level
   - Sell side: min-heap by price, FIFO within price level

   Order Types: Market (execute at best available), Limit (specific price),
                Stop-Loss (trigger at threshold)

2. Matching Engine:
   - Single-threaded per symbol (avoids locking, deterministic)
   - In-memory sorted structure (red-black tree of price levels)
   - Each price level: doubly-linked list of orders (FIFO)
   - Match: new buy order walks sell book from lowest, vice versa

3. Sequencer: All orders get a global sequence number
   Single point of serialization → deterministic replay

4. Event Sourcing: Every order/cancel/trade is an event
   Reconstruct order book state from event log at any point

5. Market Data Feed: Publish L1 (best bid/ask) and L2 (full depth)
   Multicast/WebSocket to subscribers, batched at microsecond intervals

6. Fault Tolerance: Hot standby matching engine replaying same event stream
   If primary fails, standby has identical state, promotes instantly

7. Low Latency Techniques:
   - Kernel bypass (DPDK/RDMA) for network
   - Lock-free data structures
   - Pre-allocated memory pools (no GC pauses)
   - Colocation: trader servers in same datacenter
```

---

## 17.21 Design Comparison Summary

```
System                │ Key Components                      │ Key Challenge
──────────────────────│─────────────────────────────────────│──────────────────────
URL Shortener         │ KGS, base62, cache                  │ Collision handling
Rate Limiter          │ Token bucket, Redis                  │ Distributed accuracy
Notification          │ Kafka, workers per channel           │ Delivery guarantee
Chat System           │ WebSocket, Cassandra                 │ Real-time + offline
Social Feed           │ Fan-out, timeline cache              │ Celebrity problem
Video Streaming       │ Transcoding DAG, CDN, HLS           │ Adaptive bitrate
Ride Sharing          │ GeoHash, matching algorithm         │ Real-time location
Search Engine         │ Inverted index, PageRank             │ Relevance ranking
File Storage          │ Chunking, delta sync, dedup          │ Sync conflicts
Web Crawler           │ URL frontier, Bloom filter           │ Politeness + scale
Typeahead             │ Trie, top-K per node                 │ Fresh suggestions
Payment System        │ Double-entry, idempotency            │ Exactly-once
KV Store              │ Consistent hash, quorum              │ Conflict resolution
Ticket Booking        │ Seat locking, virtual queue          │ Double booking
Task Scheduler        │ Sorted set, distributed lock         │ Exactly-once execution
Google Maps           │ Contraction Hierarchies, tile CDN    │ Real-time rerouting
E-Commerce            │ Saga, inventory lock, search         │ Flash sale consistency
Collaborative Editing │ OT/CRDT, operation log               │ Conflict resolution
Social Network        │ Graph DB, PYMK, hybrid feed          │ Graph traversal at scale
Stock Exchange        │ Order book, sequencer                │ Sub-ms matching latency
```

---

[← Previous: Distributed Systems](16-distributed-systems.md) | [Next: Design Patterns →](18-design-patterns.md)
