# Chapter 5: Advanced & Probabilistic Data Structures 🟡🔴

---

## 5.1 Disjoint Set (Union-Find) 🟢

Tracks a set of elements partitioned into non-overlapping groups. Supports two operations:
- **Find:** Which group does element x belong to?
- **Union:** Merge two groups

```
Initially:  {0} {1} {2} {3} {4}   (each element in its own set)
Union(0,1): {0,1} {2} {3} {4}
Union(2,3): {0,1} {2,3} {4}
Union(1,3): {0,1,2,3} {4}
Find(2) == Find(0)?  → Yes (same set)
Find(4) == Find(0)?  → No (different sets)
```

### Implementation with Path Compression & Union by Rank

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))  # Each element is its own parent
        self.rank = [0] * n
        self.count = n  # Number of distinct sets

    def find(self, x):
        """Find root of x with path compression — amortized O(α(n)) ≈ O(1)"""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # Path compression
        return self.parent[x]

    def union(self, x, y):
        """Merge sets containing x and y — amortized O(α(n)) ≈ O(1)"""
        root_x = self.find(x)
        root_y = self.find(y)

        if root_x == root_y:
            return False  # Already in same set

        # Union by rank — attach smaller tree under larger
        if self.rank[root_x] < self.rank[root_y]:
            self.parent[root_x] = root_y
        elif self.rank[root_x] > self.rank[root_y]:
            self.parent[root_y] = root_x
        else:
            self.parent[root_y] = root_x
            self.rank[root_x] += 1

        self.count -= 1
        return True

    def connected(self, x, y):
        return self.find(x) == self.find(y)

# Usage: Detect cycle in undirected graph
def has_cycle(edges, n):
    uf = UnionFind(n)
    for u, v in edges:
        if not uf.union(u, v):
            return True  # u and v already connected → cycle!
    return False

# Usage: Count connected components
uf = UnionFind(5)
uf.union(0, 1)
uf.union(2, 3)
print(uf.count)  # 3 sets: {0,1}, {2,3}, {4}
```

**α(n)** is the inverse Ackermann function — grows incredibly slowly, effectively O(1) for all practical inputs.

**Used in:** Kruskal's MST algorithm, network connectivity, image processing, percolation.

---

## 5.2 Skip List 🟡

A probabilistic data structure that's an alternative to balanced BSTs. Multiple levels of linked lists where higher levels skip over more elements.

```
Level 3:  HEAD ────────────────────────────── 50 ──── NIL
Level 2:  HEAD ────── 10 ──────────── 30 ──── 50 ──── NIL
Level 1:  HEAD ── 5 ── 10 ──── 20 ── 30 ──── 50 ──── NIL
Level 0:  HEAD ── 5 ── 10 ── 15 ── 20 ── 30 ── 40 ── 50 ── NIL

Search for 20:
  Start at Level 3: HEAD → 50 (too far) → go down
  Level 2: HEAD → 10 → 30 (too far) → go down from 10
  Level 1: 10 → 20 → FOUND!
```

```python
import random

class SkipNode:
    def __init__(self, key, level):
        self.key = key
        self.forward = [None] * (level + 1)

class SkipList:
    def __init__(self, max_level=16, p=0.5):
        self.max_level = max_level
        self.p = p
        self.level = 0
        self.header = SkipNode(-float('inf'), max_level)

    def random_level(self):
        level = 0
        while random.random() < self.p and level < self.max_level:
            level += 1
        return level

    def search(self, key):
        """O(log n) expected"""
        current = self.header
        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
        current = current.forward[0]
        return current and current.key == key

    def insert(self, key):
        """O(log n) expected"""
        update = [None] * (self.max_level + 1)
        current = self.header

        for i in range(self.level, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
            update[i] = current

        level = self.random_level()
        if level > self.level:
            for i in range(self.level + 1, level + 1):
                update[i] = self.header
            self.level = level

        new_node = SkipNode(key, level)
        for i in range(level + 1):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node

# All operations: O(log n) expected time
```

**Used in:** Redis sorted sets, LevelDB/RocksDB, concurrent data structures.

---

## 5.3 Bloom Filter 🟡

A space-efficient probabilistic structure that tests set membership. It can tell you:
- "Definitely NOT in set" (100% accurate)
- "Probably in set" (small false positive rate)

**No false negatives, but possible false positives.**

```
Bit array: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]  (m = 10 bits)

Insert "cat" using k=3 hash functions:
  h1("cat") = 2, h2("cat") = 5, h3("cat") = 8

  [0, 0, 1, 0, 0, 1, 0, 0, 1, 0]

Insert "dog":
  h1("dog") = 1, h2("dog") = 5, h3("dog") = 9

  [0, 1, 1, 0, 0, 1, 0, 0, 1, 1]

Check "cat": positions 2,5,8 all set → "probably yes" ✓
Check "bird": h1=3, h2=7, h3=1 → position 3 is 0 → "definitely no" ✓
Check "fox": h1=1, h2=2, h3=5 → all 1s → "probably yes"
  (FALSE POSITIVE — "fox" was never added!)
```

```python
import hashlib

class BloomFilter:
    def __init__(self, size, num_hashes):
        self.size = size
        self.num_hashes = num_hashes
        self.bits = [False] * size

    def _hashes(self, item):
        """Generate k hash values"""
        hashes = []
        for i in range(self.num_hashes):
            h = hashlib.md5(f"{item}{i}".encode()).hexdigest()
            hashes.append(int(h, 16) % self.size)
        return hashes

    def add(self, item):
        for h in self._hashes(item):
            self.bits[h] = True

    def might_contain(self, item):
        return all(self.bits[h] for h in self._hashes(item))

# Optimal parameters:
# m (bits) = -n * ln(p) / (ln(2))²
# k (hashes) = (m/n) * ln(2)
# where n = expected items, p = desired false positive rate
```

**Used in:** Spell checkers, network routers, database query optimization, Bitcoin SPV nodes, Chrome safe browsing.

---

## 5.4 Count-Min Sketch 🟡

A probabilistic structure for estimating the **frequency** of elements in a stream.

```
k hash functions, each mapping to a row of w counters:

             0   1   2   3   4   5   6   7
Row 0 (h0): [0] [0] [2] [0] [1] [0] [3] [0]
Row 1 (h1): [1] [0] [0] [3] [0] [0] [2] [0]
Row 2 (h2): [0] [3] [0] [0] [0] [2] [0] [1]

To estimate count of "x": take minimum across rows
  h0("x")=2, h1("x")=3, h2("x")=5
  estimate = min(2, 3, 2) = 2

Never underestimates, but may overestimate.
```

```python
class CountMinSketch:
    def __init__(self, width, depth):
        self.width = width
        self.depth = depth
        self.table = [[0] * width for _ in range(depth)]

    def _hash(self, item, i):
        h = hash(f"{item}_{i}")
        return h % self.width

    def add(self, item, count=1):
        for i in range(self.depth):
            self.table[i][self._hash(item, i)] += count

    def estimate(self, item):
        return min(self.table[i][self._hash(item, i)] for i in range(self.depth))
```

**Used in:** Network traffic analysis, database query planning, trending detection.

---

## 5.5 HyperLogLog 🟡

Estimates the **number of distinct elements** (cardinality) using very little memory.

```
Idea: Hash each element. Count the maximum number of leading zeros
in any hash. If you saw 5 leading zeros, you probably processed
about 2⁵ = 32 distinct elements.

Uses multiple "registers" to reduce variance.
Memory: ~1.5 KB for estimates with ~2% error on billions of elements!
```

**Used in:** Redis `PFCOUNT`, database `COUNT(DISTINCT)`, analytics.

---

## 5.6 Cuckoo Filter 🟡

An improvement over Bloom filters that supports **deletion** and is often faster.

- Stores fingerprints (short hashes) in a cuckoo hash table
- Supports: insert, lookup, delete
- Better space efficiency than Bloom filter for false positive rates < 3%

---

## 5.7 Quotient Filter 🟡

Cache-friendly alternative to Bloom filters using open addressing. Better locality than Bloom filters for on-disk storage.

---

## 5.8 Sparse Table 🟡

Precomputes answers for all ranges of length 2^k. Answers **Range Minimum Queries** (RMQ) in O(1) after O(n log n) preprocessing.

```python
import math

class SparseTable:
    def __init__(self, arr):
        n = len(arr)
        k = int(math.log2(n)) + 1
        self.table = [[0] * n for _ in range(k)]
        self.log = [0] * (n + 1)

        # Precompute logs
        for i in range(2, n + 1):
            self.log[i] = self.log[i // 2] + 1

        # Fill first row
        for i in range(n):
            self.table[0][i] = arr[i]

        # Fill remaining rows (Dynamic Programming)
        for j in range(1, k):
            for i in range(n - (1 << j) + 1):
                self.table[j][i] = min(
                    self.table[j-1][i],
                    self.table[j-1][i + (1 << (j-1))]
                )

    def query(self, l, r):
        """Range minimum in O(1)"""
        j = self.log[r - l + 1]
        return min(self.table[j][l], self.table[j][r - (1 << j) + 1])

# Usage
arr = [1, 3, 2, 7, 9, 11, 3, 5, 6, 4]
st = SparseTable(arr)
print(st.query(2, 6))  # Minimum of arr[2..6] = min(2,7,9,11,3) = 2
```

**Limitation:** Only works for idempotent operations (min, max, GCD) — NOT for sum (because overlapping ranges would double-count).

---

## 5.9 Mo's Algorithm (Sqrt Decomposition for Queries) 🟡

Answer offline range queries in O((N+Q)√N) by sorting queries cleverly.

```python
import math

def mos_algorithm(arr, queries):
    """
    Offline range query algorithm — O((N+Q)√N)

    queries: list of (left, right, query_index)
    Returns: answer for each query

    Key idea: Sort queries by (left // √N, right) to minimize
    pointer movement. Each pointer moves O(√N) per query amortized.
    """
    n = len(arr)
    block = max(1, int(math.sqrt(n)))

    # Sort queries by (block of left, right)
    sorted_queries = sorted(
        range(len(queries)),
        key=lambda i: (queries[i][0] // block,
                       queries[i][1] if (queries[i][0] // block) % 2 == 0
                       else -queries[i][1])  # Zig-zag optimization
    )

    # Current state
    cur_l, cur_r = 0, -1
    current_answer = 0
    freq = {}
    answers = [0] * len(queries)

    def add(idx):
        nonlocal current_answer
        val = arr[idx]
        freq[val] = freq.get(val, 0) + 1
        # Example: count distinct elements
        if freq[val] == 1:
            current_answer += 1

    def remove(idx):
        nonlocal current_answer
        val = arr[idx]
        freq[val] -= 1
        if freq[val] == 0:
            current_answer -= 1
            del freq[val]

    for qi in sorted_queries:
        l, r = queries[qi][0], queries[qi][1]

        # Expand/shrink to reach [l, r]
        while cur_r < r:
            cur_r += 1
            add(cur_r)
        while cur_l > l:
            cur_l -= 1
            add(cur_l)
        while cur_r > r:
            remove(cur_r)
            cur_r -= 1
        while cur_l < l:
            remove(cur_l)
            cur_l += 1

        answers[qi] = current_answer

    return answers

# Example: Count distinct elements in ranges
arr = [1, 3, 3, 4, 1, 2]
queries = [(0, 3), (1, 4), (2, 5)]  # (left, right) inclusive
result = mos_algorithm(arr, queries)
print(result)  # [3, 3, 4] — distinct counts
```

```
Sqrt Decomposition Patterns:

  1. Mo's Algorithm: Offline range queries — O((N+Q)√N)
  2. Block decomposition: Split array into √N blocks
     - Point update: O(√N), Range query: O(√N)
  3. Mo with updates: Add timestamp dimension — O((N+Q)^{2/3} · N^{1/3})
  4. Mo on trees: Flatten with Euler tour, then apply Mo's

When to use vs alternatives:
  - Segment Tree: O(log N) per query, supports updates → prefer for online
  - Sparse Table: O(1) query, no updates → prefer for static RMQ
  - Mo's: O(√N) per query, offline only → use when add/remove are cheap
    and problem doesn't fit standard DS
```

---

## 5.9 LRU Cache 🟢

**Least Recently Used** cache — when the cache is full, evict the item that hasn't been used for the longest time.

Uses a **hash map** + **doubly linked list** for O(1) access and eviction.

```
Capacity: 3

Access A → [A]
Access B → [B, A]
Access C → [C, B, A]
Access A → [A, C, B]       ← A moves to front
Access D → [D, A, C]       ← B evicted (least recently used)
```

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # Mark as recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # Remove oldest

# Manual implementation with HashMap + Doubly Linked List
class DLLNode:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCacheManual:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}
        self.head = DLLNode()  # Dummy head
        self.tail = DLLNode()  # Dummy tail
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_front(node)
            return node.val
        return -1

    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = DLLNode(key, value)
        self.cache[key] = node
        self._add_to_front(node)
        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

**Used in:** CPU caches, database buffer pools, web caches, OS page replacement.

---

## 5.10 LFU Cache 🟡

**Least Frequently Used** cache — evicts the item with the lowest access count.

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.key_val = {}
        self.key_freq = {}
        self.freq_keys = defaultdict(OrderedDict)  # freq → {key: None} in order
        self.min_freq = 0

    def get(self, key):
        if key not in self.key_val:
            return -1
        self._increase_freq(key)
        return self.key_val[key]

    def put(self, key, value):
        if self.capacity <= 0:
            return
        if key in self.key_val:
            self.key_val[key] = value
            self._increase_freq(key)
            return
        if len(self.key_val) >= self.capacity:
            # Evict LFU (ties broken by LRU)
            evict_key, _ = self.freq_keys[self.min_freq].popitem(last=False)
            del self.key_val[evict_key]
            del self.key_freq[evict_key]
        self.key_val[key] = value
        self.key_freq[key] = 1
        self.freq_keys[1][key] = None
        self.min_freq = 1

    def _increase_freq(self, key):
        freq = self.key_freq[key]
        self.key_freq[key] = freq + 1
        del self.freq_keys[freq][key]
        if not self.freq_keys[freq]:
            del self.freq_keys[freq]
            if self.min_freq == freq:
                self.min_freq += 1
        self.freq_keys[freq + 1][key] = None
```

---

## 5.11 Persistent Data Structures 🔴

Data structures that preserve all previous versions of themselves when modified.

### Types of Persistence

| Type | Description |
|------|------------|
| **Partial** | Can access any old version but only modify the latest |
| **Full** | Can access and modify any version |
| **Confluent** | Can merge two versions |
| **Functional** | Immutable — every "modification" creates a new version |

### Fat Node Method

Each node stores a list of (timestamp, value) pairs.

### Path Copying

On modification, copy only the nodes on the path from root to the modified node. All other nodes are shared.

```
Original (v0):       After update at leaf (v1):
     A                    A'           (new root)
    / \                  / \
   B   C                B   C'        (new node, shared B)
  / \                  / \
 D   E                D   E'          (new leaf)
```

**Used in:** Functional programming (Clojure, Haskell), version control, retroactive data structures, computational geometry.

---

## 5.12 Succinct Data Structures 🔴

Use space close to the **information-theoretic minimum** while still supporting efficient operations.

| Structure | Standard Space | Succinct Space |
|-----------|---------------|----------------|
| Bit vector (rank/select) | O(n) | n + o(n) bits |
| Binary tree (n nodes) | O(n) pointers | 2n + o(n) bits |
| Arbitrary tree | O(n) pointers | 2n + o(n) bits |

**Example:** A bit vector of length n storing r bits needs ⌈log₂(C(n,r))⌉ bits in theory. Succinct structures come within lower-order terms of this.

**Used in:** Compressed text indexes, bioinformatics, space-efficient databases.

---

## 5.13 Treap as Implicit Data Structure 🟡

An **implicit treap** uses array indices as keys (implicitly). This gives a balanced BST that supports:
- Insert/delete at arbitrary position: O(log n)
- Split/merge arrays: O(log n)
- Range operations: O(log n)

Think of it as a **flexible array** with logarithmic operations for everything.

---

## 5.14 Zipper 🟡

A technique from functional programming to navigate and modify immutable tree structures. It stores:
- The current focus node
- The "context" (path back to root)

```
Zipper on tree at node C:
  Focus: C
  Context: "C is right child of A, and A has left child B"

  Move left from A:
  Focus: B
  Context: "B is left child of A, and A has right child C"
```

**Used in:** Haskell, Clojure, any functional language working with trees.

---

## 5.15 Multimap & Multiset 🟢

- **Multiset (Bag):** A set that allows duplicate elements
- **Multimap:** A map that allows multiple values per key

```python
from collections import Counter, defaultdict

# Multiset using Counter
bag = Counter([1, 2, 2, 3, 3, 3])
print(bag)  # Counter({3: 3, 2: 2, 1: 1})
bag[2] += 1
print(bag[2])  # 3

# Multimap using defaultdict(list)
mm = defaultdict(list)
mm['color'].append('red')
mm['color'].append('blue')
mm['size'].append('large')
print(mm['color'])  # ['red', 'blue']
```

---

## 5.16 Trie Variants 🟡

### Bitwise Trie (Binary Trie)

Each edge represents a bit (0 or 1). Used for:
- Maximum XOR queries
- IP routing (longest prefix match)

```python
class BitwiseTrie:
    def __init__(self):
        self.root = {}

    def insert(self, num):
        node = self.root
        for i in range(31, -1, -1):
            bit = (num >> i) & 1
            if bit not in node:
                node[bit] = {}
            node = node[bit]

    def max_xor(self, num):
        node = self.root
        result = 0
        for i in range(31, -1, -1):
            bit = (num >> i) & 1
            toggled = 1 - bit
            if toggled in node:
                result |= (1 << i)
                node = node[toggled]
            else:
                node = node[bit]
        return result
```

### HAMT (Hash Array Mapped Trie) 🔴

Used in persistent data structures (Clojure, Scala). A trie where each level uses a portion of the hash code. Efficient structural sharing.

---

## 5.17 Fibonacci Search & Related Structures

**Fibonacci Search:** A search algorithm for sorted arrays using Fibonacci numbers to determine probe positions. Works well when array access has non-uniform cost.

---

## 5.18 Self-Organizing Lists 🟡

Lists that reorder themselves based on access patterns:

1. **Move to Front (MTF):** Most recently accessed element moves to front
2. **Transpose:** Accessed element swaps with its predecessor
3. **Count:** Elements ordered by access frequency

```python
class MoveToFrontList:
    def __init__(self):
        self.items = []

    def access(self, item):
        if item in self.items:
            self.items.remove(item)
            self.items.insert(0, item)  # Move to front
            return True
        return False

    def insert(self, item):
        self.items.insert(0, item)  # Insert at front
```

MTF is **2-competitive** — never more than twice as slow as the optimal static ordering.

---

## 5.19 X-fast and Y-fast Tries 🔴

Improvements over Van Emde Boas trees with better space complexity.

| Structure | Space | Search | Insert/Delete | Predecessor |
|-----------|-------|--------|---------------|-------------|
| Van Emde Boas | O(U) | O(log log U) | O(log log U) | O(log log U) |
| X-fast Trie | O(n log U) | O(log log U) | O(log U) | O(log log U) |
| Y-fast Trie | O(n) | O(log log U) | O(log log U) | O(log log U) |

---

## 5.20 Dancing Links (DLX) 🔴

A technique for implementing Algorithm X (exact cover solver) using doubly linked lists. Efficiently covers and uncovers columns.

**Used in:** Sudoku solvers, pentomino puzzles, exact cover problems.

```python
# The key insight: removing and restoring nodes in a doubly linked list
def cover(node):
    node.left.right = node.right
    node.right.left = node.left

def uncover(node):
    node.left.right = node
    node.right.left = node
    # This restore works because node still has its original pointers!
```

---

## 5.21 Finger Trees 🔴

A functional data structure supporting:
- O(1) amortized access to both ends
- O(log n) concatenation and splitting
- Can implement sequences, priority queues, and search trees

**Used in:** Haskell's `Data.Sequence`.

---

## 5.22 LSM-Tree (Log-Structured Merge Tree) 🟡

Write-optimized tree powering LevelDB, RocksDB, Cassandra, BigTable.

```
Write path:
  1. Write to in-memory sorted buffer (memtable)
  2. When full, flush to disk as sorted file (SSTable)
  3. Background compaction merges SSTables

Read path:
  1. Check memtable (memory)
  2. Check Bloom filters for each level
  3. Search SSTables from newest to oldest

         ┌─────────┐
         │ Memtable │ ← Writes go here (memory)
         └────┬─────┘
              ↓ flush
         ┌─────────┐
         │ Level 0  │ ← Small SSTables
         └────┬─────┘
              ↓ compaction
         ┌─────────┐
         │ Level 1  │ ← Larger, merged
         └────┬─────┘
              ↓
         ┌─────────┐
         │ Level 2  │ ← Even larger
         └─────────┘

Write: O(1) amortized     (append to log)
Read:  O(log n) worst      (check multiple levels)
Space amplification: ~1.1× to 2×
```

---

## 5.23 Adaptive Radix Tree (ART) 🔴

Space-efficient trie with adaptive node sizes: 4, 16, 48, or 256 children.

```
Node4:    ≤4 children — linear scan (fits in cache line)
Node16:   ≤16 children — SIMD parallel search
Node48:   ≤48 children — 256-byte index + 48 child pointers
Node256:  full array (like standard trie node)

Faster than hash tables for in-memory key-value stores.
Used in HyPER database engine.
```

---

## 5.24 Xor Filter 🟡

Newer alternative to Bloom/Cuckoo filters — faster lookups, more space-efficient.

```python
# Xor filter concept:
# Store fingerprints in 3 hash locations such that:
#   table[h0(x)] ⊕ table[h1(x)] ⊕ table[h2(x)] = fingerprint(x)
# Lookup: check if XOR of 3 positions equals fingerprint
# False positive rate ~1/2^b for b-bit fingerprints

# ~1.23 bits per key for 0.4% FP rate (vs ~1.44 for Bloom)
# Build is O(n), lookup is O(1) with 3 memory accesses
# Drawback: static (no dynamic inserts/deletes)
```

---

## 5.25 Retroactive Data Structures 🔴

Modify **past operations** and see the effect on the present. Generalizes persistence.

```
Timeline:  t1     t2     t3     t4     t5
           insert  insert  delete  insert  query
           (3)     (7)     (3)     (5)     → {7, 5}

Retroactive: go back to t2, insert(10)
New present: {7, 10, 5}

Partial retroactive: query only current state
Full retroactive: query any past state
```

---

## Summary: When to Use What

| Need | Data Structure |
|------|---------------|
| Fast set membership with small memory | Bloom Filter / Xor Filter |
| Fast frequency estimation | Count-Min Sketch |
| Count distinct elements | HyperLogLog |
| Dynamic connectivity / grouping | Union-Find |
| O(1) range minimum query | Sparse Table |
| Cache with eviction | LRU / LFU Cache |
| Alternative to balanced BST | Skip List |
| Version history of data | Persistent structures |
| Minimum memory representation | Succinct structures |
| IP routing / XOR queries | Bitwise Trie |
| Write-heavy key-value store | LSM-Tree |
| In-memory string indexing | Adaptive Radix Tree |
| Modify past operations | Retroactive DS |

---

[← Previous: Hash Tables, Heaps & Graphs](04-data-structures-hash-heap-graph.md) | [Next: Sorting & Searching →](06-sorting-and-searching.md)
