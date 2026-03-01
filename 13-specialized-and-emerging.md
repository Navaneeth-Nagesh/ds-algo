# Chapter 13: Specialized, Parallel & Emerging Algorithms 🔴🔮

---

# Part A: Compression Algorithms

## 13.1 Run-Length Encoding (RLE) 🟢

Replace consecutive repeats with count + character.

```python
def rle_encode(s):
    if not s: return ""
    result = []
    count = 1
    for i in range(1, len(s)):
        if s[i] == s[i-1]:
            count += 1
        else:
            result.append(f"{count}{s[i-1]}")
            count = 1
    result.append(f"{count}{s[-1]}")
    return "".join(result)

def rle_decode(s):
    result = []
    i = 0
    while i < len(s):
        num = ""
        while i < len(s) and s[i].isdigit():
            num += s[i]
            i += 1
        result.append(s[i] * int(num))
        i += 1
    return "".join(result)

print(rle_encode("AAABBBCCDDDDDD"))  # "3A3B2C6D"
```

---

## 13.2 Huffman Coding 🟡

See [Chapter 10](10-greedy-backtracking-divide-conquer.md) for full implementation. Optimal prefix-free code based on character frequencies.

---

## 13.3 LZW Compression 🟡

Dictionary-based compression used in GIF, TIFF, Unix compress.

```python
def lzw_compress(data):
    # Initialize dictionary with single characters
    dictionary = {chr(i): i for i in range(256)}
    next_code = 256

    result = []
    w = ""
    for c in data:
        wc = w + c
        if wc in dictionary:
            w = wc
        else:
            result.append(dictionary[w])
            dictionary[wc] = next_code
            next_code += 1
            w = c
    if w:
        result.append(dictionary[w])
    return result

def lzw_decompress(compressed):
    dictionary = {i: chr(i) for i in range(256)}
    next_code = 256

    result = [chr(compressed[0])]
    w = result[0]

    for code in compressed[1:]:
        if code in dictionary:
            entry = dictionary[code]
        elif code == next_code:
            entry = w + w[0]
        else:
            raise ValueError("Invalid compressed data")

        result.append(entry)
        dictionary[next_code] = w + entry[0]
        next_code += 1
        w = entry

    return "".join(result)

data = "TOBEORNOTTOBEORTOBEORNOT"
compressed = lzw_compress(data)
print(f"Original: {len(data)} chars, Compressed: {len(compressed)} codes")
print(lzw_decompress(compressed) == data)  # True
```

---

## 13.4 Arithmetic Coding 🔴

Encodes entire message as a single number in [0, 1). More efficient than Huffman for skewed distributions.

```python
def arithmetic_encode(message, freq):
    """Simplified arithmetic encoding"""
    # Build cumulative frequency table
    total = sum(freq.values())
    cum_freq = {}
    low = 0
    for char in sorted(freq.keys()):
        cum_freq[char] = (low / total, (low + freq[char]) / total)
        low += freq[char]

    lo, hi = 0.0, 1.0
    for char in message:
        range_size = hi - lo
        char_lo, char_hi = cum_freq[char]
        hi = lo + range_size * char_hi
        lo = lo + range_size * char_lo

    return (lo + hi) / 2  # Any value in [lo, hi) works
```

---

# Part B: Randomized Algorithms

## 13.5 Reservoir Sampling 🟢

Select k random items from a stream of unknown size.

```python
import random

def reservoir_sampling(stream, k):
    reservoir = []
    for i, item in enumerate(stream):
        if i < k:
            reservoir.append(item)
        else:
            j = random.randint(0, i)
            if j < k:
                reservoir[j] = item
    return reservoir
```

---

## 13.6 Monte Carlo Methods 🟢

Randomized algorithms that may give incorrect results with bounded probability.

```python
# Estimate π using Monte Carlo
def estimate_pi(n):
    inside = 0
    for _ in range(n):
        x = random.random()
        y = random.random()
        if x*x + y*y <= 1:
            inside += 1
    return 4 * inside / n

# Randomized primality testing (Miller-Rabin) → See Chapter 11
```

---

## 13.7 Las Vegas Algorithms 🟢

Always give correct results, but runtime is random.

```python
# Randomized Quick Sort — expected O(n log n)
def randomized_quicksort(arr):
    if len(arr) <= 1: return arr
    pivot = random.choice(arr)
    left = [x for x in arr if x < pivot]
    mid = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return randomized_quicksort(left) + mid + randomized_quicksort(right)

# Randomized Quick Select — expected O(n)
def quick_select(arr, k):
    pivot = random.choice(arr)
    left = [x for x in arr if x < pivot]
    mid = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]

    if k <= len(left):
        return quick_select(left, k)
    elif k <= len(left) + len(mid):
        return pivot
    else:
        return quick_select(right, k - len(left) - len(mid))
```

---

## 13.8 Skip List (Randomized Data Structure) 🟡

See [Chapter 5](05-data-structures-advanced.md) for full implementation.

Uses randomized level assignment for O(log n) expected operations.

---

# Part C: Parallel and Distributed Algorithms

## 13.9 MapReduce Pattern 🟡

```python
from functools import reduce
from collections import Counter

# Word count with MapReduce
def map_phase(documents):
    """Map: document → (word, 1) pairs"""
    pairs = []
    for doc in documents:
        for word in doc.split():
            pairs.append((word.lower(), 1))
    return pairs

def shuffle_phase(pairs):
    """Shuffle: group by key"""
    groups = {}
    for key, value in pairs:
        groups.setdefault(key, []).append(value)
    return groups

def reduce_phase(groups):
    """Reduce: aggregate values per key"""
    return {key: sum(values) for key, values in groups.items()}

# Simulated MapReduce
docs = ["hello world", "hello python", "world of python"]
pairs = map_phase(docs)
groups = shuffle_phase(pairs)
result = reduce_phase(groups)
print(result)  # {'hello': 2, 'world': 2, 'python': 2, 'of': 1}
```

---

## 13.10 Parallel Prefix Sum (Scan) 🟡

```python
# Sequential prefix sum
def prefix_sum(arr):
    result = [0] * len(arr)
    result[0] = arr[0]
    for i in range(1, len(arr)):
        result[i] = result[i-1] + arr[i]
    return result

# Parallel prefix sum concept (Blelloch scan)
# Up-sweep: reduce pairs → O(log n) steps
# Down-sweep: compute all prefixes → O(log n) steps
# Total: O(n) work, O(log n) steps with n/2 processors
```

---

## 13.11 Parallel Sorting 🟡

```python
# Bitonic Sort — parallel-friendly, O(log²n) steps with n processors
def bitonic_sort(arr, ascending=True):
    def compare_and_swap(arr, i, j, ascending):
        if (arr[i] > arr[j]) == ascending:
            arr[i], arr[j] = arr[j], arr[i]

    def bitonic_merge(arr, lo, cnt, ascending):
        if cnt > 1:
            k = cnt // 2
            for i in range(lo, lo + k):
                compare_and_swap(arr, i, i + k, ascending)
            bitonic_merge(arr, lo, k, ascending)
            bitonic_merge(arr, lo + k, k, ascending)

    def bitonic_sort_rec(arr, lo, cnt, ascending):
        if cnt > 1:
            k = cnt // 2
            bitonic_sort_rec(arr, lo, k, True)
            bitonic_sort_rec(arr, lo + k, k, False)
            bitonic_merge(arr, lo, cnt, ascending)

    n = len(arr)
    # Pad to power of 2
    next_pow2 = 1
    while next_pow2 < n:
        next_pow2 <<= 1
    padded = arr + [float('inf')] * (next_pow2 - n)
    bitonic_sort_rec(padded, 0, next_pow2, ascending)
    return padded[:n]

# Parallel Merge Sort: split array, sort halves in parallel, merge
# Work: O(n log n), Steps: O(log²n) with n processors
```

---

## 13.12 Concurrent Data Structures 🟡

```python
import threading

# Lock-based concurrent hash map (conceptual)
class ConcurrentHashMap:
    def __init__(self, num_buckets=16):
        self.num_buckets = num_buckets
        self.buckets = [[] for _ in range(num_buckets)]
        self.locks = [threading.Lock() for _ in range(num_buckets)]

    def _hash(self, key):
        return hash(key) % self.num_buckets

    def put(self, key, value):
        idx = self._hash(key)
        with self.locks[idx]:
            for i, (k, v) in enumerate(self.buckets[idx]):
                if k == key:
                    self.buckets[idx][i] = (key, value)
                    return
            self.buckets[idx].append((key, value))

    def get(self, key):
        idx = self._hash(key)
        with self.locks[idx]:
            for k, v in self.buckets[idx]:
                if k == key:
                    return v
        return None

# Lock-free concepts:
# - Compare-And-Swap (CAS): atomic check-and-update
# - Lock-free queue: Michael & Scott queue
# - Lock-free stack: Treiber stack
# - Read-Copy-Update (RCU): readers never block
```

---

# Part D: Cache-Oblivious Algorithms

## 13.13 Cache-Oblivious Concepts 🔴

Algorithms that perform well regardless of cache size, without knowing the cache parameters.

```
Memory Hierarchy:
  L1 Cache (fast, small) → L2 → L3 → RAM (slow, large) → Disk

Cache-oblivious algorithms:
  ┌──────────────────────────────────────────────┐
  │ Van Emde Boas layout for trees               │
  │ Cache-oblivious matrix multiplication         │
  │ Funnel sort                                   │
  │ Cache-oblivious B-tree                        │
  │ Fractal tree index                            │
  └──────────────────────────────────────────────┘
```

```python
# Cache-oblivious matrix multiplication (recursive divide)
def co_matrix_mult(A, B, C, n, ax, ay, bx, by, cx, cy):
    """
    C[cx:cx+n][cy:cy+n] += A[ax:ax+n][ay:ay+n] × B[bx:bx+n][by:by+n]
    Recursively divides into 8 sub-problems
    """
    if n <= 64:  # Base case fits in cache
        for i in range(n):
            for j in range(n):
                for k in range(n):
                    C[cx+i][cy+j] += A[ax+i][ay+k] * B[bx+k][by+j]
        return

    m = n // 2
    co_matrix_mult(A, B, C, m, ax, ay, bx, by, cx, cy)
    co_matrix_mult(A, B, C, m, ax, ay+m, bx+m, by, cx, cy)
    co_matrix_mult(A, B, C, m, ax, ay, bx, by+m, cx, cy+m)
    co_matrix_mult(A, B, C, m, ax, ay+m, bx+m, by+m, cx, cy+m)
    co_matrix_mult(A, B, C, m, ax+m, ay, bx, by, cx+m, cy)
    co_matrix_mult(A, B, C, m, ax+m, ay+m, bx+m, by, cx+m, cy)
    co_matrix_mult(A, B, C, m, ax+m, ay, bx, by+m, cx+m, cy+m)
    co_matrix_mult(A, B, C, m, ax+m, ay+m, bx+m, by+m, cx+m, cy+m)
```

---

# Part E: Streaming Algorithms

## 13.14 Streaming & Sublinear Algorithms 🔴

Process data in a single pass with limited memory.

```python
# Count-Min Sketch → See Chapter 5
# HyperLogLog → See Chapter 5
# Bloom Filter → See Chapter 5

# Misra-Gries Frequent Items (heavy hitters)
def frequent_items(stream, k):
    """Find items appearing more than n/k times"""
    counters = {}
    for item in stream:
        if item in counters:
            counters[item] += 1
        elif len(counters) < k - 1:
            counters[item] = 1
        else:
            to_remove = []
            for key in counters:
                counters[key] -= 1
                if counters[key] == 0:
                    to_remove.append(key)
            for key in to_remove:
                del counters[key]
    return list(counters.keys())

# Flajolet-Martin distinct count
def fm_distinct_count(stream):
    max_trailing_zeros = 0
    for item in stream:
        h = hash(item) & 0xFFFFFFFF
        if h == 0: continue
        trailing = (h & -h).bit_length() - 1
        max_trailing_zeros = max(max_trailing_zeros, trailing)
    return 2 ** max_trailing_zeros
```

---

# Part F: Cryptographic Algorithms

## 13.15 Hashing Algorithms 🟡

```python
import hashlib

# SHA-256
def sha256_hash(data):
    return hashlib.sha256(data.encode()).hexdigest()

# MD5 (not cryptographically secure, still used for checksums)
def md5_hash(data):
    return hashlib.md5(data.encode()).hexdigest()

# Simple hash function (educational)
def simple_hash(data, mod=2**32):
    h = 0
    for byte in data.encode():
        h = (h * 31 + byte) % mod
    return h
```

## 13.16 Symmetric Encryption Concepts 🟡

```python
# XOR cipher (simplest symmetric encryption)
def xor_encrypt(plaintext, key):
    encrypted = []
    for i, char in enumerate(plaintext):
        encrypted.append(chr(ord(char) ^ ord(key[i % len(key)])))
    return "".join(encrypted)

def xor_decrypt(ciphertext, key):
    return xor_encrypt(ciphertext, key)  # XOR is its own inverse

# AES, DES, ChaCha20 — use libraries like `cryptography` in production
```

## 13.17 Asymmetric Encryption (RSA Concept) 🟡

```python
# Simplified RSA (educational only — real RSA uses large primes)
def simple_rsa_demo():
    # Key generation
    p, q = 61, 53
    n = p * q               # 3233
    phi = (p - 1) * (q - 1) # 3120
    e = 17                   # Public exponent (coprime to phi)
    d = mod_inverse(e, phi)  # 2753 — private exponent

    # Encrypt
    message = 42
    ciphertext = pow(message, e, n)  # 42^17 mod 3233 = 2557

    # Decrypt
    plaintext = pow(ciphertext, d, n)  # 2557^2753 mod 3233 = 42

    print(f"Public key: (e={e}, n={n})")
    print(f"Private key: (d={d}, n={n})")
    print(f"Message: {message} → Encrypted: {ciphertext} → Decrypted: {plaintext}")

def mod_inverse(a, m):
    def extended_gcd(a, b):
        if a == 0: return b, 0, 1
        g, x1, y1 = extended_gcd(b % a, a)
        return g, y1 - (b // a) * x1, x1
    _, x, _ = extended_gcd(a % m, m)
    return x % m
```

---

# Part G: Machine Learning Foundations

## 13.18 Core ML Algorithms (as CS Algorithms) 🟡

```python
import math

# k-Nearest Neighbors
def knn(train_X, train_y, query, k=3):
    distances = []
    for i, x in enumerate(train_X):
        d = math.sqrt(sum((a - b) ** 2 for a, b in zip(x, query)))
        distances.append((d, train_y[i]))
    distances.sort()

    # Majority vote among k nearest
    from collections import Counter
    votes = Counter(label for _, label in distances[:k])
    return votes.most_common(1)[0][0]

# k-Means Clustering
def kmeans(points, k, max_iter=100):
    import random
    centroids = random.sample(points, k)

    for _ in range(max_iter):
        # Assign points to nearest centroid
        clusters = [[] for _ in range(k)]
        for p in points:
            dists = [sum((a-b)**2 for a, b in zip(p, c)) for c in centroids]
            clusters[dists.index(min(dists))].append(p)

        # Update centroids
        new_centroids = []
        for cluster in clusters:
            if cluster:
                centroid = [sum(dim) / len(cluster) for dim in zip(*cluster)]
                new_centroids.append(centroid)
            else:
                new_centroids.append(random.choice(points))

        if new_centroids == centroids:
            break
        centroids = new_centroids

    return centroids, clusters

# Gradient Descent
def gradient_descent(f_grad, x0, learning_rate=0.01, epochs=1000):
    x = x0
    for _ in range(epochs):
        grad = f_grad(x)
        x = [xi - learning_rate * gi for xi, gi in zip(x, grad)]
    return x

# Decision Tree (ID3 simplified)
def entropy(labels):
    from collections import Counter
    counts = Counter(labels)
    total = len(labels)
    return -sum((c/total) * math.log2(c/total) for c in counts.values())
```

---

# Part H: Quantum Algorithms

## 13.19 Quantum Computing Concepts 🔮

```
Classical bit:  0 or 1
Quantum bit (qubit): α|0⟩ + β|1⟩  where |α|² + |β|² = 1

Key concepts:
  Superposition — qubit in multiple states simultaneously
  Entanglement — qubits correlated across distance
  Interference — amplitudes can add or cancel
  Measurement — collapses superposition to 0 or 1
```

### Shor's Algorithm (Integer Factoring) 🔮

- Factors integers in O((log n)³) — exponentially faster than classical
- Breaks RSA encryption
- Uses quantum Fourier transform + period finding

### Grover's Algorithm (Unstructured Search) 🔮

- Searches unsorted database of N items in O(√N) — quadratic speedup
- Optimal for unstructured search

### Quantum Approximate Optimization (QAOA) 🔮

- Solves combinatorial optimization problems
- Hybrid classical-quantum approach

```python
# Simulating Grover's-like search classically (conceptual)
def grover_conceptual(database, target):
    """
    Quantum Grover's searches in O(√N).
    Classical equivalent for illustration.
    """
    import math
    N = len(database)
    iterations = int(math.pi / 4 * math.sqrt(N))

    # In real quantum computing:
    # 1. Initialize all qubits in superposition
    # 2. Apply oracle (marks target state)
    # 3. Apply diffusion operator (amplifies marked state)
    # 4. Repeat √N times
    # 5. Measure

    print(f"Classical: O({N}) lookups needed")
    print(f"Quantum:   O({iterations}) iterations (√{N} ≈ {iterations})")

    # Classical fallback
    for i, item in enumerate(database):
        if item == target:
            return i
    return -1
```

---

# Part I: Emerging & Future Algorithms

## 13.20 Learned Data Structures 🔮

Traditional indexes replaced by ML models that learn data distribution.

```
Learned Index:
  Traditional B-tree: O(log n) lookup
  Learned index: ML model predicts position → O(1) average with correction

  Model: CDF(key) → approximate position
  Then linear search in small neighborhood

Learned Bloom Filter:
  ML classifier replaces or augments traditional Bloom filter
  Fewer false positives with same memory
```

## 13.21 Succinct & Compressed Data Structures 🔮

```
Store data in space close to information-theoretic minimum
while supporting efficient queries.

  Succinct bitvector: n + o(n) bits, O(1) rank/select
  Wavelet tree: O(n log σ) bits for sequences
  FM-index: compressed full-text index (used in bioinformatics)
  Grammar-compressed strings: O(g) representation where g ≪ n
```

## 13.22 Lattice-Based Cryptography 🔮

Post-quantum cryptographic algorithms resistant to quantum computers.

```
NTRU, CRYSTALS-Kyber, CRYSTALS-Dilithium
Based on hard lattice problems:
  - Learning With Errors (LWE)
  - Shortest Vector Problem (SVP)

These are being standardized by NIST as quantum-safe replacements
for RSA and ECC.
```

## 13.23 Differential Privacy Algorithms 🔮

```python
import random, math

# Laplace Mechanism — add noise to preserve privacy
def laplace_mechanism(true_value, sensitivity, epsilon):
    """
    Add Laplace noise for ε-differential privacy
    sensitivity: max change in output from one person's data
    epsilon: privacy budget (smaller = more private)
    """
    scale = sensitivity / epsilon
    noise = random.random() - 0.5
    noise = -scale * math.copysign(1, noise) * math.log(1 - 2 * abs(noise))
    return true_value + noise
```

## 13.24 Homomorphic Encryption 🔮

Compute on encrypted data without decrypting. Enables privacy-preserving cloud computing.

## 13.25 Blockchain & Consensus Algorithms 🔮

```
Proof of Work (Bitcoin): miners solve hash puzzles
Proof of Stake (Ethereum 2.0): validators stake coins
PBFT: Byzantine fault-tolerant consensus for permissioned networks
Raft: leader-based consensus for distributed systems
```

## 13.26 Federated Learning 🔮

Train ML models across decentralized data without sharing raw data.

---

# Part J: Algorithm Design Meta-Strategies

## 13.27 Online Algorithms & Competitive Analysis 🟡

Algorithms that process input piece-by-piece without knowing the future.

```python
# Secretary Problem (Optimal Stopping)
# Hire the best candidate when you must decide immediately after each interview.
import math, random

def secretary_problem(candidates):
    """
    Optimal strategy: Reject first n/e candidates,
    then pick first one better than all rejected.
    Success probability → 1/e ≈ 36.8%
    """
    n = len(candidates)
    reject = max(1, int(n / math.e))

    # Phase 1: Observe and reject first n/e
    threshold = max(candidates[:reject])

    # Phase 2: Pick first candidate better than threshold
    for i in range(reject, n):
        if candidates[i] > threshold:
            return i, candidates[i]

    return n - 1, candidates[-1]  # Forced to pick last

# Ski Rental Problem
def ski_rental_optimal(days_skiing, buy_cost, rent_cost):
    """
    Deterministic: Buy on day ⌈buy_cost/rent_cost⌉ → competitive ratio = 2
    Randomized: Buy on random day ~ competitive ratio = e/(e-1) ≈ 1.58
    """
    breakeven = buy_cost // rent_cost
    total_rent = min(days_skiing, breakeven) * rent_cost
    if days_skiing > breakeven:
        return total_rent + buy_cost  # Rent then buy
    return total_rent

# Online Paging (LRU is k-competitive)
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity
        self.faults = 0

    def access(self, page):
        if page in self.cache:
            self.cache.move_to_end(page)
            return False  # Cache hit
        else:
            self.faults += 1
            if len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)  # Evict LRU
            self.cache[page] = True
            return True  # Cache miss (page fault)

# LRU is k-competitive where k = cache size
# Meaning: LRU faults ≤ k × OPT faults (for any sequence)
```

```
Competitive Analysis Summary:
  Algorithm          | Competitive Ratio | Problem
  -------------------|-------------------|------------------
  LRU Paging         | k                 | Online paging
  FIFO Paging        | k                 | Online paging
  Marking Algorithm  | k (randomized: Hₖ)| Online paging
  Secretary (det.)   | 1/e ≈ 0.368       | Optimal stopping
  Ski Rental (det.)  | 2                 | Rent vs buy
  Ski Rental (rand.) | e/(e-1) ≈ 1.58   | Rent vs buy
  List Accessing     | 2 (Move-to-Front) | Self-organizing lists
  Load Balancing     | 2 - 1/m (greedy)  | Online load balancing
```

## 13.28 Approximation Algorithms 🟡

Polynomial-time algorithms with provable approximation guarantees for NP-hard problems.

```python
# Vertex Cover: 2-Approximation
def vertex_cover_2approx(n, edges):
    """
    Greedy matching: pick any edge, add both endpoints.
    Guaranteed: |solution| ≤ 2 × |OPT|
    Time: O(V + E)
    """
    covered = set()
    cover = set()

    for u, v in edges:
        if u not in covered and v not in covered:
            cover.add(u)
            cover.add(v)
            covered.add(u)
            covered.add(v)

    return cover

# Set Cover: O(log n)-Approximation (Greedy)
def greedy_set_cover(universe, sets):
    """
    Greedy: always pick set that covers most uncovered elements.
    Approximation ratio: H(max_set_size) ≈ ln(n) + 1
    """
    uncovered = set(universe)
    selected = []
    remaining = [s for s in sets]

    while uncovered:
        # Pick set covering most uncovered elements
        best = max(remaining, key=lambda s: len(s & uncovered))
        selected.append(best)
        uncovered -= best
        remaining.remove(best)

    return selected

# Christofides' Algorithm for TSP: 1.5-Approximation
def christofides_conceptual(graph):
    """
    For metric TSP (triangle inequality holds):
    1. Find MST
    2. Find minimum-weight perfect matching on odd-degree MST vertices
    3. Combine MST + matching → Euler circuit
    4. Shortcut repeated vertices → Hamiltonian cycle

    Guarantee: tour ≤ 1.5 × OPT
    """
    pass  # Implementation requires matching library

# Knapsack FPTAS (Fully Polynomial-Time Approximation Scheme)
def knapsack_fptas(weights, values, capacity, epsilon):
    """
    (1-ε)-approximation for knapsack in O(n² / ε) time.
    Scale down values to reduce DP state space.
    """
    n = len(weights)
    v_max = max(values)

    # Scale factor
    K = epsilon * v_max / n

    # Scaled values (round down)
    scaled = [int(v / K) for v in values]

    # Standard DP on scaled values
    max_scaled = sum(scaled)
    dp = [float('inf')] * (max_scaled + 1)
    dp[0] = 0

    for i in range(n):
        for v in range(max_scaled, scaled[i] - 1, -1):
            dp[v] = min(dp[v], dp[v - scaled[i]] + weights[i])

    # Find best feasible
    best = 0
    for v in range(max_scaled + 1):
        if dp[v] <= capacity:
            best = max(best, v)

    return best * K  # Unscale

# PTAS vs FPTAS
"""
PTAS: For any fixed ε > 0, runs in poly(n) time (may be exponential in 1/ε)
FPTAS: Runs in poly(n, 1/ε) time — fully polynomial!

Examples:
  - Knapsack: has FPTAS
  - Bin Packing: has PTAS (no FPTAS unless P=NP)
  - Euclidean TSP: has PTAS (Arora, Mitchell)
  - General TSP: NO constant-factor approximation (unless P=NP)
  - MAX-3SAT: 7/8-approximation (random assignment!)
"""
```

### LP Relaxation & Rounding 🔴

```
Integer Linear Program (ILP) → NP-hard in general
LP Relaxation: Allow fractional variables → solve in polynomial time
Rounding: Convert fractional solution to integer solution

Example: Weighted Vertex Cover
  Minimize Σ wᵢxᵢ  subject to  xᵢ + xⱼ ≥ 1 for each edge (i,j),  xᵢ ∈ {0,1}

  LP relaxation: allow 0 ≤ xᵢ ≤ 1
  Rounding: xᵢ ≥ 0.5 → include vertex i

  This gives 2-approximation for weighted vertex cover.

Other LP-based techniques:
  - Randomized rounding
  - Primal-dual method
  - Semidefinite programming (SDP) relaxation for MAX-CUT (Goemans-Williamson: 0.878-approx)
```

## 13.29 Game Theory Algorithms 🟡

```python
# Minimax with Alpha-Beta Pruning
def minimax(state, depth, alpha, beta, maximizing, evaluate, get_moves, apply_move, undo_move):
    """
    Minimax with alpha-beta pruning — O(b^(d/2)) with good ordering vs O(b^d).

    evaluate(state): heuristic score
    get_moves(state): list of legal moves
    apply_move/undo_move: modify state
    """
    if depth == 0 or not get_moves(state):
        return evaluate(state), None

    best_move = None

    if maximizing:
        value = float('-inf')
        for move in get_moves(state):
            apply_move(state, move)
            child_value, _ = minimax(state, depth-1, alpha, beta, False,
                                     evaluate, get_moves, apply_move, undo_move)
            undo_move(state, move)
            if child_value > value:
                value = child_value
                best_move = move
            alpha = max(alpha, value)
            if alpha >= beta:
                break  # Beta cutoff
        return value, best_move
    else:
        value = float('inf')
        for move in get_moves(state):
            apply_move(state, move)
            child_value, _ = minimax(state, depth-1, alpha, beta, True,
                                     evaluate, get_moves, apply_move, undo_move)
            undo_move(state, move)
            if child_value < value:
                value = child_value
                best_move = move
            beta = min(beta, value)
            if alpha >= beta:
                break  # Alpha cutoff
        return value, best_move

# Monte Carlo Tree Search (MCTS)
import random, math

class MCTSNode:
    def __init__(self, state, parent=None, move=None):
        self.state = state
        self.parent = parent
        self.move = move
        self.children = []
        self.visits = 0
        self.wins = 0
        self.untried_moves = None

    def ucb1(self, c=1.41):
        """Upper Confidence Bound"""
        if self.visits == 0:
            return float('inf')
        return self.wins / self.visits + c * math.sqrt(math.log(self.parent.visits) / self.visits)

def mcts(root_state, iterations, get_moves, apply_move, simulate, get_result):
    """
    Four phases repeated:
    1. Selection: Walk tree using UCB1 to find most promising node
    2. Expansion: Add new child node
    3. Simulation: Random playout from new node
    4. Backpropagation: Update win/visit counts up the tree
    """
    root = MCTSNode(root_state)
    root.untried_moves = get_moves(root_state)

    for _ in range(iterations):
        node = root
        state = root_state.copy() if hasattr(root_state, 'copy') else root_state

        # Selection
        while not node.untried_moves and node.children:
            node = max(node.children, key=lambda n: n.ucb1())
            state = apply_move(state, node.move)

        # Expansion
        if node.untried_moves:
            move = random.choice(node.untried_moves)
            node.untried_moves.remove(move)
            state = apply_move(state, move)
            child = MCTSNode(state, parent=node, move=move)
            child.untried_moves = get_moves(state)
            node.children.append(child)
            node = child

        # Simulation (random playout)
        sim_state = state
        result = simulate(sim_state)

        # Backpropagation
        while node:
            node.visits += 1
            node.wins += result
            node = node.parent

    # Return most-visited child's move
    return max(root.children, key=lambda n: n.visits).move

# Expectimax (for games with chance nodes)
def expectimax(state, depth, maximizing, evaluate, get_moves, get_chances):
    """For games with randomness (e.g., 2048, Backgammon)"""
    if depth == 0:
        return evaluate(state)

    if maximizing:
        return max(expectimax(apply(state, m), depth-1, False, evaluate, get_moves, get_chances)
                   for m in get_moves(state))
    else:
        # Chance node: expected value over random outcomes
        chances = get_chances(state)
        return sum(prob * expectimax(s, depth-1, True, evaluate, get_moves, get_chances)
                   for s, prob in chances)
```

## 13.30 Metaheuristics 🟡

General-purpose optimization strategies for hard combinatorial problems.

```python
import random, math

# Simulated Annealing
def simulated_annealing(initial, neighbor, cost, temp=1000, cooling=0.995, min_temp=1e-8):
    """
    Probabilistically accept worse solutions to escape local optima.
    P(accept worse) = exp(-ΔE / T) — decreases as T cools.
    """
    current = initial
    current_cost = cost(current)
    best = current
    best_cost = current_cost
    T = temp

    while T > min_temp:
        candidate = neighbor(current)
        candidate_cost = cost(candidate)
        delta = candidate_cost - current_cost

        if delta < 0 or random.random() < math.exp(-delta / T):
            current = candidate
            current_cost = candidate_cost
            if current_cost < best_cost:
                best = current
                best_cost = current_cost

        T *= cooling

    return best, best_cost

# Genetic Algorithm
def genetic_algorithm(pop_size, gene_length, fitness, mutate, crossover,
                      generations=1000, elite_frac=0.1):
    """
    1. Initialize random population
    2. Evaluate fitness
    3. Select parents (tournament/roulette)
    4. Crossover to create children
    5. Mutate children
    6. Repeat
    """
    # Initialize
    population = [[random.randint(0,1) for _ in range(gene_length)]
                  for _ in range(pop_size)]

    for gen in range(generations):
        # Evaluate
        scores = [(fitness(ind), ind) for ind in population]
        scores.sort(reverse=True)

        # Elite selection
        n_elite = max(1, int(pop_size * elite_frac))
        new_pop = [ind for _, ind in scores[:n_elite]]

        # Create rest via crossover + mutation
        while len(new_pop) < pop_size:
            # Tournament selection
            p1 = max(random.sample(scores, 3))[1]
            p2 = max(random.sample(scores, 3))[1]
            child = crossover(p1, p2)
            child = mutate(child)
            new_pop.append(child)

        population = new_pop

    best = max(population, key=fitness)
    return best, fitness(best)

# Tabu Search
def tabu_search(initial, neighbors, cost, tabu_tenure=10, max_iter=1000):
    """
    Like hill climbing but maintains tabu list to avoid cycling.
    """
    current = initial
    best = current
    best_cost = cost(current)
    tabu_list = []

    for _ in range(max_iter):
        candidates = [(n, cost(n)) for n in neighbors(current)
                      if n not in tabu_list]
        if not candidates:
            break

        # Best non-tabu neighbor (aspiration: allow if better than best)
        candidates.sort(key=lambda x: x[1])
        current, current_cost = candidates[0]

        if current_cost < best_cost:
            best = current
            best_cost = current_cost

        tabu_list.append(current)
        if len(tabu_list) > tabu_tenure:
            tabu_list.pop(0)

    return best, best_cost

# Ant Colony Optimization (ACO) — for TSP-like problems
def ant_colony_tsp(dist, n_ants=20, n_iter=100, alpha=1, beta=2, rho=0.5):
    """
    Pheromone-based search: ants build solutions probabilistically,
    deposit pheromone on good paths.
    """
    n = len(dist)
    pheromone = [[1.0]*n for _ in range(n)]
    best_tour = None
    best_length = float('inf')

    for _ in range(n_iter):
        tours = []
        for _ in range(n_ants):
            tour = [random.randint(0, n-1)]
            visited = set(tour)
            for _ in range(n - 1):
                current = tour[-1]
                # Probability of choosing next city
                probs = []
                for j in range(n):
                    if j not in visited:
                        tau = pheromone[current][j] ** alpha
                        eta = (1.0 / max(dist[current][j], 1e-10)) ** beta
                        probs.append((j, tau * eta))
                    else:
                        probs.append((j, 0))

                total = sum(p for _, p in probs)
                if total == 0:
                    break
                r = random.random() * total
                cumulative = 0
                for j, p in probs:
                    cumulative += p
                    if cumulative >= r:
                        tour.append(j)
                        visited.add(j)
                        break

            length = sum(dist[tour[i]][tour[i+1]] for i in range(len(tour)-1))
            length += dist[tour[-1]][tour[0]]
            tours.append((tour, length))

            if length < best_length:
                best_length = length
                best_tour = tour

        # Evaporate pheromone
        for i in range(n):
            for j in range(n):
                pheromone[i][j] *= (1 - rho)

        # Deposit pheromone
        for tour, length in tours:
            deposit = 1.0 / length
            for i in range(len(tour) - 1):
                pheromone[tour[i]][tour[i+1]] += deposit
                pheromone[tour[i+1]][tour[i]] += deposit

    return best_tour, best_length

# Particle Swarm Optimization (PSO)
def particle_swarm(objective, bounds, n_particles=30, n_iter=100, w=0.7, c1=1.5, c2=1.5):
    """Continuous optimization using swarm intelligence"""
    dim = len(bounds)

    # Initialize particles
    particles = []
    for _ in range(n_particles):
        pos = [random.uniform(lo, hi) for lo, hi in bounds]
        vel = [random.uniform(-(hi-lo)*0.1, (hi-lo)*0.1) for lo, hi in bounds]
        particles.append({'pos': pos, 'vel': vel,
                         'best_pos': pos[:], 'best_val': objective(pos)})

    global_best = min(particles, key=lambda p: p['best_val'])
    gbest_pos = global_best['best_pos'][:]
    gbest_val = global_best['best_val']

    for _ in range(n_iter):
        for p in particles:
            for d in range(dim):
                r1, r2 = random.random(), random.random()
                p['vel'][d] = (w * p['vel'][d] +
                              c1 * r1 * (p['best_pos'][d] - p['pos'][d]) +
                              c2 * r2 * (gbest_pos[d] - p['pos'][d]))
                p['pos'][d] += p['vel'][d]
                p['pos'][d] = max(bounds[d][0], min(bounds[d][1], p['pos'][d]))

            val = objective(p['pos'])
            if val < p['best_val']:
                p['best_val'] = val
                p['best_pos'] = p['pos'][:]
                if val < gbest_val:
                    gbest_val = val
                    gbest_pos = p['pos'][:]

    return gbest_pos, gbest_val
```

```
Metaheuristic Comparison:
  Method                   | Best For                    | Nature Inspired?
  -------------------------|-----------------------------|-----------------
  Simulated Annealing      | Single-trajectory search    | Metal cooling
  Genetic Algorithm        | Combinatorial optimization  | Evolution
  Tabu Search              | Neighborhood search         | No (memory-based)
  Ant Colony               | Routing / TSP               | Ant foraging
  Particle Swarm           | Continuous optimization     | Bird flocking
  Hill Climbing            | Simple local search         | No
  Variable Neighborhood    | Escaping local optima       | No
```

## 13.31 External Memory Algorithms 🟡

Algorithms optimized for disk I/O, where memory is limited.

```
I/O Model (Aggarwal-Vitter):
  M = main memory size (in elements)
  B = disk block size
  N = problem size

  Scanning: O(N/B) I/Os
  Sorting: O((N/B) log_{M/B}(N/B)) I/Os  ← not O(N log N)!
  B-tree search: O(log_B N) I/Os

  Key insight: sequential access ≫ random access

  External Merge Sort:
    1. Read M elements → sort in memory → write sorted run
    2. Merge M/B - 1 runs at a time
    3. Each merge pass: O(N/B) I/Os
    4. Number of passes: O(log_{M/B}(N/M))

  Buffer Tree:
    Batched B-tree operations. Each operation costs O((1/B) log_{M/B}(N/B)) amortized I/Os.

  External Hashing:
    Linear probing with blocks. O(1) expected I/Os per operation.
```

## 13.32 Additional Randomized Algorithms 🟡

```python
# Karger's Minimum Cut (Randomized Contraction)
def karger_min_cut(n, edges, iterations=None):
    """
    Randomly contract edges until 2 vertices remain.
    Repeat O(n² log n) times for high probability of finding min cut.
    Success probability per trial: ≥ 2/n²
    """
    import random

    if iterations is None:
        iterations = n * n * 2  # High confidence

    min_cut = float('inf')

    for _ in range(iterations):
        # Union-Find for contraction
        parent = list(range(n))

        def find(x):
            while parent[x] != x:
                parent[x] = parent[parent[x]]
                x = parent[x]
            return x

        def union(x, y):
            parent[find(x)] = find(y)

        remaining = n
        edge_list = list(edges)
        random.shuffle(edge_list)

        for u, v in edge_list:
            if remaining <= 2:
                break
            if find(u) != find(v):
                union(u, v)
                remaining -= 1

        # Count crossing edges
        cut_size = sum(1 for u, v in edges if find(u) != find(v))
        min_cut = min(min_cut, cut_size)

    return min_cut

# Random Walk on Graph
def random_walk_cover_time(adj, start=0, max_steps=100000):
    """Expected time to visit all vertices via random walk.
    Cover time ≤ O(n³) for any connected graph, O(n log n) for complete graph."""
    n = len(adj)
    visited = {start}
    current = start
    steps = 0

    while len(visited) < n and steps < max_steps:
        current = random.choice(adj[current])
        visited.add(current)
        steps += 1

    return steps

# Randomized Rounding
def randomized_rounding_set_cover(universe, sets, costs):
    """
    1. Solve LP relaxation to get fractional solution x*
    2. Include set S_i independently with probability x*_i
    3. Repeat O(log n) times to cover all elements
    Expected cost ≤ O(log n) × OPT
    """
    pass  # Requires LP solver
```

## 13.33 Concurrent Data Structures (Detailed) 🟡

```python
import threading

# Lock-Free Stack (Treiber Stack)
class TreiberStack:
    """Lock-free stack using compare-and-swap (simulated with lock)"""
    def __init__(self):
        self._head = None
        self._lock = threading.Lock()  # Simulating CAS

    def push(self, value):
        new_node = [value, None]
        while True:
            with self._lock:
                new_node[1] = self._head
                self._head = new_node
                return

    def pop(self):
        while True:
            with self._lock:
                head = self._head
                if head is None:
                    return None
                self._head = head[1]
                return head[0]

# Michael-Scott Lock-Free Queue
class MSQueue:
    """Lock-free queue (simplified — real version uses CAS)"""
    def __init__(self):
        sentinel = [None, None]
        self._head = sentinel
        self._tail = sentinel
        self._lock = threading.Lock()

    def enqueue(self, value):
        new_node = [value, None]
        with self._lock:
            self._tail[1] = new_node
            self._tail = new_node

    def dequeue(self):
        with self._lock:
            if self._head[1] is None:
                return None
            value = self._head[1][0]
            self._head = self._head[1]
            return value

"""
Concurrency Levels (weakest to strongest):
  1. Obstruction-free: guaranteed progress if run alone
  2. Lock-free: at least one thread makes progress
  3. Wait-free: every thread makes progress in bounded steps

Read-Copy-Update (RCU):
  - Readers read without locks (zero overhead)
  - Writers create new copy, atomically swap pointer
  - Old copy freed after all readers finish (grace period)
  - Used extensively in Linux kernel

Work Stealing:
  - Each thread has a deque of tasks
  - Push/pop own tasks from one end
  - Steal tasks from other threads' deques when idle
  - Used in: Java ForkJoinPool, Cilk, Intel TBB
"""
```

---

## 13.34 When to Use What?

```
Decision Tree for Algorithm Selection:

Is the problem a known type?
├── Yes → Use standard algorithm
└── No → Analyze structure
         ├── Optimal substructure + overlapping subproblems → DP
         ├── Optimal substructure + greedy choice → Greedy
         ├── Independent subproblems → Divide and Conquer
         ├── Need all solutions → Backtracking
         ├── Constraint satisfaction → Backtracking + pruning / AC-3
         ├── NP-hard (need exact) → Branch and Bound / ILP
         ├── NP-hard (approx OK) → Approximation Algorithm
         ├── NP-hard (no guarantee needed) → Metaheuristic
         ├── Online / streaming → Online algorithms / streaming DS
         ├── Massive dataset → External memory / MapReduce
         ├── Real-time constraints → Amortized / online algorithms
         ├── Game / adversarial → Minimax / MCTS / Alpha-Beta
         └── Continuous optimization → Gradient descent / PSO / SA
```

## 13.35 Future Directions

| Area | Impact |
|------|--------|
| **Quantum algorithms** | Exponential speedup for specific problems |
| **Learned data structures** | Replace general-purpose with data-aware |
| **Neuromorphic computing** | Brain-inspired parallel processing |
| **DNA computing** | Massive parallelism for combinatorial problems |
| **Optical computing** | Speed-of-light matrix operations |
| **Approximate computing** | Trade accuracy for massive performance |
| **Self-adjusting structures** | Automatically optimize for access patterns |
| **Persistent & functional DS** | Immutable structures for concurrent systems |
| **Graph neural networks** | ML on graph-structured data |
| **Differentiable programming** | Gradient-based optimization of algorithms |

---

## Complete Algorithm Taxonomy

```
Algorithms & Data Structures
├── Data Structures
│   ├── Linear: Array, Linked List, Stack, Queue, Deque, Difference Array, Bit Array
│   ├── Trees: BST, AVL, Red-Black, B-Tree, Trie, Segment, Fenwick, Splay,
│   │          Treap, K-D, AA, 2-3-4, Ball, R-Tree, Quadtree, Range Tree, ...
│   ├── Heaps: Binary, Fibonacci, Binomial, Pairing, Soft, Leftist, ...
│   ├── Hash: Hash Table, Bloom Filter, Cuckoo, Count-Min Sketch, Quotient, Xor, ...
│   ├── Graphs: Adjacency List/Matrix, Edge List, CSR/CSC
│   └── Advanced: Skip List, Union-Find, Sparse Table, LSM-Tree, ART, ...
├── Sorting
│   ├── Comparison: Merge, Quick, Heap, Insertion, Tim, Intro, Block, Tournament, ...
│   └── Non-comparison: Counting, Radix, Bucket, Sample, American Flag, ...
├── Searching: Linear, Binary, Ternary, Interpolation, IDA*, IDDFS, Beam, B&B, ...
├── Graph: BFS, DFS, Dijkstra, Bellman-Ford, Floyd-Warshall, Kruskal, Prim,
│          Tarjan, Kosaraju, Hopcroft-Karp, 2-SAT, PageRank, Blossom, ...
├── DP: Knapsack, LCS, LIS, Edit Distance, Interval, Bitmask, Tree, Digit,
│       Probability, Game (Sprague-Grundy), CHT/Li Chao, Monotone Queue, Steiner, ...
├── String: KMP, Rabin-Karp, Boyer-Moore, Z, Aho-Corasick, Suffix Array/Tree,
│           Suffix Automaton, Manacher, Eertree, Bitap, FM-Index, Lyndon, ...
├── Greedy: Activity Selection, Huffman, Matroid, Weighted Interval, Set Cover, ...
├── Divide & Conquer: Merge Sort, Strassen, Karatsuba, Closest Pair, ...
├── Backtracking: N-Queens, Sudoku, Subsets, Knight's Tour, Algorithm X/DLX, ...
├── Math: GCD, Sieve, FFT/NTT, Matrix Exp, CRT, Möbius, Burnside, Lagrange,
│         Simplex, Baby-step Giant-step, Newton's, Simpson's, Stirling, ...
├── Bit Manipulation: XOR tricks, Gray Code, Bitmask enumeration, ...
├── Geometry: Convex Hull (Graham, Andrew, Chan), Sweep Line, Voronoi, Delaunay,
│             Pick's, Welzl's, Minkowski Sum, Half-Plane Intersection, ...
├── Network Flow: Ford-Fulkerson, Dinic, Push-Relabel, König's, Min-Cost, ...
├── Approximation: Vertex Cover, Set Cover, Christofides, FPTAS, LP Rounding, ...
├── Online: Secretary, Ski Rental, Paging (LRU), Competitive Analysis, ...
├── Game Theory: Minimax, Alpha-Beta, MCTS, Expectimax, Sprague-Grundy, ...
├── Metaheuristics: SA, GA, Tabu, ACO, PSO, Hill Climbing, VNS, ...
├── Randomized: Monte Carlo, Las Vegas, Reservoir Sampling, Karger's Cut, ...
├── Parallel: MapReduce, Bitonic Sort, Parallel Prefix, Work Stealing, ...
├── Streaming: Count-Min Sketch, HyperLogLog, Heavy Hitters, ...
├── Compression: Huffman, LZW, RLE, Arithmetic, BWT, ...
├── Cryptographic: RSA, AES, SHA, Lattice-based, ...
├── External Memory: External Sort, B-Tree, Buffer Tree, ...
├── Concurrent: Lock-Free (Treiber, MS Queue), RCU, Wait-Free, ...
└── Emerging: Quantum (Shor, Grover), Learned Indexes, Federated Learning, ...
```

---

[← Previous: Network Flow & Geometry](12-network-flow-and-geometry.md) | [Back to Index →](README.md)
