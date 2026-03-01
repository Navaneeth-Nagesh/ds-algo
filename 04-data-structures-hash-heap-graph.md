# Chapter 4: Hash Tables, Heaps & Graph Representations 🟢🟡

---

## Part A: Hash Tables

## 4.1 What is a Hash Table?

A hash table (hash map) stores **key-value pairs** using a hash function to compute an index into an array of buckets.

```
Key → hash(key) → index → bucket
                          ┌─────────┐
"apple"  → hash → 2  →   │         │  bucket 0
"banana" → hash → 5  →   │         │  bucket 1
"cherry" → hash → 2  →   │ apple ──┼→ cherry   bucket 2 (collision!)
                          │         │  bucket 3
                          │         │  bucket 4
                          │ banana  │  bucket 5
                          └─────────┘
```

### Operations & Complexity

| Operation | Average | Worst |
|-----------|---------|-------|
| Insert | O(1) | O(n) |
| Search | O(1) | O(n) |
| Delete | O(1) | O(n) |

Worst case occurs when all keys hash to the same bucket.

### Hash Functions

A good hash function:
1. **Deterministic** — same input always gives same output
2. **Uniform distribution** — spreads keys evenly across buckets
3. **Fast to compute** — should be O(1) or O(k) where k is key size

```python
# Simple hash function for strings
def simple_hash(key, table_size):
    hash_val = 0
    for char in key:
        hash_val = (hash_val * 31 + ord(char)) % table_size
    return hash_val

# Python's built-in hash
print(hash("hello"))  # Some integer
print(hash(42))       # 42 (integers hash to themselves)

# Common hash functions in practice:
# - MurmurHash (fast, good distribution)
# - CityHash (by Google, optimized for strings)
# - SipHash (Python 3.4+, resistant to DoS attacks)
# - xxHash (extremely fast)
# - SHA-256 (cryptographic, slower but collision-resistant)
```

---

## 4.2 Collision Resolution

### Method 1: Separate Chaining

Each bucket holds a linked list (or other structure) of all entries that hash to that bucket.

```
Bucket 0: → (key1, val1) → (key5, val5) → None
Bucket 1: → (key2, val2) → None
Bucket 2: → (key3, val3) → (key4, val4) → (key6, val6) → None
Bucket 3: → None
```

```python
class HashTableChaining:
    def __init__(self, capacity=16):
        self.capacity = capacity
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]
        self.load_factor_threshold = 0.75

    def _hash(self, key):
        return hash(key) % self.capacity

    def put(self, key, value):
        idx = self._hash(key)
        bucket = self.buckets[idx]

        # Update if key exists
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)
                return

        # Insert new key
        bucket.append((key, value))
        self.size += 1

        # Resize if load factor exceeded
        if self.size / self.capacity > self.load_factor_threshold:
            self._resize()

    def get(self, key):
        idx = self._hash(key)
        for k, v in self.buckets[idx]:
            if k == key:
                return v
        raise KeyError(key)

    def delete(self, key):
        idx = self._hash(key)
        bucket = self.buckets[idx]
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket.pop(i)
                self.size -= 1
                return
        raise KeyError(key)

    def _resize(self):
        old_buckets = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0
        for bucket in old_buckets:
            for key, value in bucket:
                self.put(key, value)
```

### Method 2: Open Addressing

All entries stored directly in the array. On collision, **probe** for the next empty slot.

#### Linear Probing

```
Insert "cat" → hash = 2, slot 2 taken → try 3 → empty → insert at 3

  0: "dog"
  1: ___
  2: "bird"
  3: "cat"    ← inserted here (hash was 2, probed to 3)
  4: ___
```

```python
class HashTableLinearProbing:
    def __init__(self, capacity=16):
        self.capacity = capacity
        self.size = 0
        self.keys = [None] * capacity
        self.values = [None] * capacity
        self.DELETED = object()  # Sentinel for deleted slots

    def _hash(self, key):
        return hash(key) % self.capacity

    def put(self, key, value):
        if self.size >= self.capacity * 0.7:
            self._resize()

        idx = self._hash(key)
        while self.keys[idx] is not None and self.keys[idx] is not self.DELETED:
            if self.keys[idx] == key:
                self.values[idx] = value  # Update
                return
            idx = (idx + 1) % self.capacity

        self.keys[idx] = key
        self.values[idx] = value
        self.size += 1

    def get(self, key):
        idx = self._hash(key)
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                return self.values[idx]
            idx = (idx + 1) % self.capacity
        raise KeyError(key)

    def delete(self, key):
        idx = self._hash(key)
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                self.keys[idx] = self.DELETED
                self.values[idx] = None
                self.size -= 1
                return
            idx = (idx + 1) % self.capacity
        raise KeyError(key)
```

#### Quadratic Probing

Instead of checking slots 1, 2, 3, ... away, check 1², 2², 3², ... away.

```
probe(i) = (hash + i²) % capacity
```

Reduces **primary clustering** (long runs of occupied slots).

#### Double Hashing

Use a second hash function to determine the probe step size.

```
probe(i) = (hash1(key) + i * hash2(key)) % capacity
```

Best distribution of all open addressing methods.

### Method 3: Cuckoo Hashing 🟡

Uses **two hash functions** and **two tables**. Each key can be in exactly one of two positions.

```
Table 1 (h1):     Table 2 (h2):
  0: "apple"        0: "date"
  1: ___             1: "banana"
  2: "cherry"        2: ___
  3: ___             3: "elderberry"

Insert "fig": h1("fig")=2 → occupied by "cherry"
              → kick "cherry" to Table 2
              → h2("cherry")=1 → occupied by "banana"
              → kick "banana" to Table 1
              → h1("banana")=0 → occupied by "apple"
              → kick "apple" to Table 2
              → h2("apple")=2 → empty → done!
```

**Guarantee:** O(1) worst-case lookup (check exactly 2 positions).

### Method 4: Robin Hood Hashing 🟡

When inserting, if the new key has traveled farther from its home than the occupying key, **swap them**. This equalizes probe lengths.

### Method 5: Hopscotch Hashing 🟡

Each bucket has a "neighborhood" of H slots. Keys must be within H slots of their hash position. Combines benefits of linear probing and chaining.

---

## 4.3 Load Factor & Resizing

```
Load Factor = number of entries / number of buckets

Typical thresholds:
  Chaining:        resize at load factor > 0.75  (Java HashMap)
  Open Addressing: resize at load factor > 0.5   (more sensitive to crowding)
```

Resizing doubles the capacity and rehashes all entries → O(n) but amortized O(1) per insertion.

---

## 4.4 Consistent Hashing 🟡

Used in distributed systems to minimally remap keys when nodes are added/removed.

```
Hash ring: 0 ─────────────────────────── 2³²
           Node A (pos 100)
           Node B (pos 300)
           Node C (pos 700)

Key "foo" → hash = 250 → goes to Node B (next node clockwise)
Key "bar" → hash = 500 → goes to Node C

If Node B removed: only keys between A and B need remapping
(instead of rehashing everything)
```

**Used in:** Amazon DynamoDB, Apache Cassandra, Content Delivery Networks.

---

## 4.5 Hash Set

A hash table that only stores **keys** (no values). Used for fast membership testing.

```python
# Python set is a hash set
s = {1, 2, 3}
s.add(4)       # O(1)
s.remove(2)    # O(1)
print(3 in s)  # O(1) → True
```

---

## 4.6 OrderedDict / LinkedHashMap

A hash map that **remembers insertion order** by maintaining a doubly linked list of entries.

```python
from collections import OrderedDict

od = OrderedDict()
od['a'] = 1
od['b'] = 2
od['c'] = 3
for key in od:
    print(key)  # a, b, c (insertion order)
```

---

# Part B: Heaps

## 4.7 What is a Heap?

A heap is a **complete binary tree** satisfying the heap property:
- **Min-Heap:** parent ≤ children (root is minimum)
- **Max-Heap:** parent ≥ children (root is maximum)

```
Min-Heap:           Max-Heap:
     1                  9
    / \                / \
   3   5              7   8
  / \                / \
 7   9              3   5
```

### Array Representation

A heap is stored as an array where:
- Parent of node at index i: `(i - 1) // 2`
- Left child: `2 * i + 1`
- Right child: `2 * i + 2`

```
Min-Heap array: [1, 3, 5, 7, 9]
Index:           0  1  2  3  4

         1 (index 0)
        / \
       3   5 (indices 1, 2)
      / \
     7   9 (indices 3, 4)
```

### Operations

| Operation | Time |
|-----------|------|
| Insert (push) | O(log n) |
| Extract min/max | O(log n) |
| Peek min/max | O(1) |
| Build heap | O(n) |
| Delete arbitrary | O(log n) |

### Implementation

```python
class MinHeap:
    def __init__(self):
        self.heap = []

    def push(self, val):
        self.heap.append(val)
        self._sift_up(len(self.heap) - 1)

    def pop(self):
        if not self.heap:
            raise IndexError("Heap is empty")
        self._swap(0, len(self.heap) - 1)
        val = self.heap.pop()
        if self.heap:
            self._sift_down(0)
        return val

    def peek(self):
        return self.heap[0]

    def _sift_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.heap[i] < self.heap[parent]:
                self._swap(i, parent)
                i = parent
            else:
                break

    def _sift_down(self, i):
        n = len(self.heap)
        while True:
            smallest = i
            left = 2 * i + 1
            right = 2 * i + 2

            if left < n and self.heap[left] < self.heap[smallest]:
                smallest = left
            if right < n and self.heap[right] < self.heap[smallest]:
                smallest = right

            if smallest != i:
                self._swap(i, smallest)
                i = smallest
            else:
                break

    def _swap(self, i, j):
        self.heap[i], self.heap[j] = self.heap[j], self.heap[i]

# Using Python's heapq module (recommended)
import heapq

pq = []
heapq.heappush(pq, 5)
heapq.heappush(pq, 1)
heapq.heappush(pq, 3)
print(heapq.heappop(pq))  # 1 (minimum)

# Build heap from list in O(n)
arr = [5, 3, 8, 1, 2]
heapq.heapify(arr)  # arr is now [1, 2, 8, 5, 3]

# Top-k elements
heapq.nlargest(3, [5, 3, 8, 1, 2])   # [8, 5, 3]
heapq.nsmallest(3, [5, 3, 8, 1, 2])  # [1, 2, 3]
```

---

## 4.8 d-ary Heap 🟡

A generalization where each node has **d children** instead of 2.

```
3-ary heap (d=3):
          1
        / | \
       3  5   2
      /|\ |\
     7 9 8 6 4
```

- **Shallower** tree → faster `decrease_key` (O(log_d n))
- **Wider** → slower `extract_min` (must compare d children)
- Optimal d depends on the use case (d=4 often good for Dijkstra)

---

## 4.9 Binomial Heap 🟡

A collection of **binomial trees** satisfying min-heap property.

```
Binomial trees:
B0: ○     B1: ○      B2:  ○       B3:    ○
               |          / \          / | \
               ○         ○   ○        ○  ○  ○
                         |            |  |
                         ○            ○  ○
                                      |
                                      ○

Bk has 2^k nodes and height k
```

| Operation | Time |
|-----------|------|
| Insert | O(log n) amortized O(1) |
| Find min | O(log n) or O(1) with pointer |
| Extract min | O(log n) |
| **Merge** | **O(log n)** — much faster than binary heap! |
| Decrease key | O(log n) |
| Delete | O(log n) |

---

## 4.10 Fibonacci Heap 🔴

The theoretically most efficient heap for certain operations.

| Operation | Amortized Time |
|-----------|---------------|
| Insert | O(1) |
| Find min | O(1) |
| Extract min | O(log n) |
| **Decrease key** | **O(1)** — key advantage! |
| Merge | O(1) |
| Delete | O(log n) |

The O(1) decrease-key makes Dijkstra's algorithm O(V log V + E) instead of O((V+E) log V).

**Structure:** A collection of trees (like binomial heap) but with a lazier approach — trees are consolidated only during `extract_min`.

**In practice:** The constant factors are large, so Fibonacci heaps are rarely used in practice. Binary heaps or Pairing heaps are usually faster for real workloads.

---

## 4.11 Pairing Heap 🟡

A simpler alternative to Fibonacci heaps with good practical performance.

```
Structure: a general tree (each node can have any number of children)

          2
        / | \
       5  3  8
      / \   |
     7   9  4
```

| Operation | Amortized Time |
|-----------|---------------|
| Insert | O(1) |
| Find min | O(1) |
| Extract min | O(log n) |
| Decrease key | O(log n) amortized (conjectured O(1)) |
| Merge | O(1) |

**In practice:** Often the fastest heap implementation.

---

## 4.12 Leftist Heap 🟡

A min-heap that supports efficient merging by maintaining a "rank" (shortest path to a null node). The left child's rank is always ≥ right child's rank.

Merge: O(log n)

---

## 4.13 Skew Heap 🟡

A self-adjusting leftist heap. On every merge, swap left and right children unconditionally. Simpler than leftist heap, amortized O(log n) merge.

---

## 4.14 Brodal Queue 🔴

Achieves the same theoretical bounds as Fibonacci heap with **worst-case** (not amortized) guarantees. Purely of theoretical interest.

---

# Part C: Graph Representations

## 4.15 What is a Graph?

A graph G = (V, E) consists of:
- **V** = set of vertices (nodes)
- **E** = set of edges (connections)

```
Undirected:          Directed:           Weighted:
  1 --- 2            1 → 2               1 --5-- 2
  |     |            ↑   ↓               |       |
  3 --- 4            3 ← 4               3 --2-- 4
                                               3
```

### Types of Graphs

| Type | Description |
|------|------------|
| **Undirected** | Edges have no direction |
| **Directed (Digraph)** | Edges have direction (u → v) |
| **Weighted** | Edges have weights/costs |
| **Unweighted** | All edges equal weight |
| **Cyclic** | Contains at least one cycle |
| **Acyclic** | No cycles (DAG if directed) |
| **Connected** | Path exists between every pair of vertices |
| **Disconnected** | Some vertices unreachable from others |
| **Complete** | Edge between every pair of vertices |
| **Bipartite** | Vertices can be split into 2 groups with no intra-group edges |
| **Sparse** | E << V² |
| **Dense** | E ≈ V² |
| **Planar** | Can be drawn with no crossing edges |
| **Multigraph** | Multiple edges between same vertices |
| **Hypergraph** | Edges can connect more than 2 vertices |

---

## 4.16 Adjacency Matrix

A 2D array where `matrix[i][j] = 1` (or weight) if there's an edge from i to j.

```
Graph:          Matrix:
  0 --- 1         0  1  2  3
  |     |     0 [ 0  1  1  0 ]
  2 --- 3     1 [ 1  0  0  1 ]
              2 [ 1  0  0  1 ]
              3 [ 0  1  1  0 ]
```

```python
class GraphMatrix:
    def __init__(self, num_vertices):
        self.V = num_vertices
        self.matrix = [[0] * num_vertices for _ in range(num_vertices)]

    def add_edge(self, u, v, weight=1):
        self.matrix[u][v] = weight
        self.matrix[v][u] = weight  # Remove for directed graph

    def has_edge(self, u, v):
        return self.matrix[u][v] != 0

    def get_neighbors(self, u):
        return [v for v in range(self.V) if self.matrix[u][v] != 0]

g = GraphMatrix(4)
g.add_edge(0, 1)
g.add_edge(0, 2)
g.add_edge(1, 3)
g.add_edge(2, 3)
```

| Operation | Time |
|-----------|------|
| Check edge exists | O(1) |
| Add edge | O(1) |
| Remove edge | O(1) |
| Get all neighbors | O(V) |
| Space | O(V²) |

**Best for:** Dense graphs, weighted graphs, small graphs, algorithms needing edge lookup.

---

## 4.17 Adjacency List

Each vertex stores a list of its neighbors.

```
Graph:           List:
  0 --- 1        0: [1, 2]
  |     |        1: [0, 3]
  2 --- 3        2: [0, 3]
                 3: [1, 2]
```

```python
from collections import defaultdict

class GraphList:
    def __init__(self, directed=False):
        self.adj = defaultdict(list)
        self.directed = directed

    def add_edge(self, u, v, weight=1):
        self.adj[u].append((v, weight))
        if not self.directed:
            self.adj[v].append((u, weight))

    def has_edge(self, u, v):
        return any(neighbor == v for neighbor, _ in self.adj[u])

    def get_neighbors(self, u):
        return self.adj[u]

g = GraphList()
g.add_edge(0, 1)
g.add_edge(0, 2)
g.add_edge(1, 3)
g.add_edge(2, 3)
```

| Operation | Time |
|-----------|------|
| Check edge exists | O(degree(u)) |
| Add edge | O(1) |
| Remove edge | O(degree(u)) |
| Get all neighbors | O(degree(u)) |
| Space | O(V + E) |

**Best for:** Sparse graphs, most graph algorithms, real-world graphs.

---

## 4.18 Edge List

Simply a list of all edges.

```python
edges = [
    (0, 1, 5),   # (from, to, weight)
    (0, 2, 3),
    (1, 3, 1),
    (2, 3, 7),
]
```

**Best for:** Kruskal's algorithm, simple edge-based processing.

---

## 4.19 Incidence Matrix

A V×E matrix where entry [v][e] = 1 if vertex v is incident to edge e.

```
Edges: e0=(0,1), e1=(0,2), e2=(1,3), e3=(2,3)

       e0  e1  e2  e3
   0 [  1   1   0   0 ]
   1 [  1   0   1   0 ]
   2 [  0   1   0   1 ]
   3 [  0   0   1   1 ]
```

Rarely used in practice. Space: O(V × E).

---

## 4.21 Universal Hashing 🟡

Choose hash function randomly from a family to guarantee low collision probability against any input.

```python
import random

class UniversalHash:
    """Carter-Wegman universal hash family: h(k) = ((a*k + b) mod p) mod m"""
    def __init__(self, m, p=2**61 - 1):
        self.m = m
        self.p = p
        self.a = random.randint(1, p - 1)
        self.b = random.randint(0, p - 1)

    def hash(self, key):
        return ((self.a * key + self.b) % self.p) % self.m

# For any two distinct keys x, y:
# Pr[h(x) = h(y)] ≤ 1/m  (same as random!)
# No adversary can construct bad inputs since the function is chosen randomly.
```

---

## 4.22 Perfect Hashing (FKS) 🟡

For a **static** key set, achieve O(1) worst-case lookup with O(n) space.

```
Two-level scheme:
  Level 1: Universal hash into m = n buckets
  Level 2: Each bucket i with nᵢ keys uses a table of size nᵢ²
           (birthday paradox → no collisions with high probability)

Total space: E[Σ nᵢ²] = O(n) when m = n

Lookup: O(1) worst case (not amortized!)
Build: O(n) expected time
```

---

## 4.23 CSR / CSC (Compressed Sparse Row/Column) 🟡

Compact graph storage for sparse graphs/matrices. Standard in scientific computing.

```python
class CSRGraph:
    """Compressed Sparse Row representation"""
    def __init__(self, n, edges):
        # edges = [(u, v, w), ...]
        from collections import defaultdict
        adj = defaultdict(list)
        for u, v, w in edges:
            adj[u].append((v, w))

        self.row_ptr = [0]   # Start index of each vertex's neighbors
        self.col_idx = []    # Destination vertices
        self.values = []     # Edge weights

        for u in range(n):
            for v, w in sorted(adj[u]):
                self.col_idx.append(v)
                self.values.append(w)
            self.row_ptr.append(len(self.col_idx))

    def neighbors(self, u):
        start, end = self.row_ptr[u], self.row_ptr[u + 1]
        return list(zip(self.col_idx[start:end], self.values[start:end]))

# Very cache-friendly; used in scipy.sparse, graph analytics frameworks
```

---

## 4.24 Soft Heap 🔴

Approximate priority queue allowing some keys to be "corrupted" (increased). Achieves O(1) amortized for insert and O(α(n)) for delete-min.

Used to achieve the **optimal deterministic minimum spanning tree** algorithm.

---

## 4.25 Heap Application Problems 🟢

### Merge K Sorted Lists

```python
import heapq

def merge_k_sorted(lists):
    """Merge k sorted lists into one sorted list. O(N log k)."""
    heap = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))

    result = []
    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)
        if elem_idx + 1 < len(lists[list_idx]):
            heapq.heappush(heap, (lists[list_idx][elem_idx + 1], list_idx, elem_idx + 1))
    return result
```

### Top-K Frequent Elements

```python
from collections import Counter

def top_k_frequent(nums, k):
    """Return k most frequent elements. O(n log k)."""
    counts = Counter(nums)
    return heapq.nlargest(k, counts.keys(), key=counts.get)

# O(n) bucket sort approach
def top_k_frequent_linear(nums, k):
    counts = Counter(nums)
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in counts.items():
        buckets[freq].append(num)
    result = []
    for i in range(len(buckets) - 1, -1, -1):
        for num in buckets[i]:
            result.append(num)
            if len(result) == k:
                return result
    return result
```

### Find Median from Data Stream

```python
class MedianFinder:
    """Two-heap approach: O(log n) add, O(1) median."""
    def __init__(self):
        self.lo = []   # max-heap (inverted) — smaller half
        self.hi = []   # min-heap — larger half

    def add_num(self, num: int):
        heapq.heappush(self.lo, -num)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))

    def find_median(self) -> float:
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2.0

# Usage: mf = MedianFinder(); mf.add_num(1); mf.add_num(2); mf.find_median() → 1.5
```

### K Closest Points to Origin

```python
def k_closest(points, k):
    """O(n log k) using max-heap of size k."""
    heap = []
    for x, y in points:
        dist = -(x*x + y*y)  # negate for max-heap
        if len(heap) < k:
            heapq.heappush(heap, (dist, x, y))
        elif dist > heap[0][0]:
            heapq.heapreplace(heap, (dist, x, y))
    return [[x, y] for _, x, y in heap]
```

---

## 4.20 Comparison of Representations

| | Adjacency Matrix | Adjacency List | Edge List | CSR |
|---|---|---|---|---|
| Space | O(V²) | O(V + E) | O(E) | O(V + E) |
| Add edge | O(1) | O(1) | O(1) | N/A (static) |
| Check edge | O(1) | O(deg) | O(E) | O(log deg) |
| Iterate neighbors | O(V) | O(deg) | O(E) | O(deg) |
| Cache performance | Good | Poor | Poor | Excellent |
| Best for | Dense, edge check | Sparse, traversal | Kruskal's, simple | Analytics, scientific |

### Rule of Thumb

- **Sparse graph** (E << V²): Use adjacency list or CSR
- **Dense graph** (E ≈ V²): Use adjacency matrix
- **Most real-world graphs** are sparse → adjacency list
- **Static graph analytics** (PageRank, BFS at scale) → CSR

---

[← Previous: Trees](03-data-structures-trees.md) | [Next: Advanced Data Structures →](05-data-structures-advanced.md)
