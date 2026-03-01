# Chapter 1: Complexity Analysis & Fundamentals 🟢

## Why Complexity Analysis Matters

Before diving into any data structure or algorithm, you need to speak the language used to compare them. Complexity analysis tells you:
- **How fast** an algorithm runs (Time Complexity)
- **How much memory** it uses (Space Complexity)
- **How it scales** as input grows

Without this, you can't make informed decisions about which tool to use.

---

## 1.1 Big O Notation (Upper Bound)

**What it means:** "In the worst case, my algorithm will never be slower than this."

Big O gives you the **upper bound** — the maximum time/space your algorithm will ever need.

### The Common Complexities (Ranked Best to Worst)

```
O(1)        →  Constant      →  Array access by index
O(log n)    →  Logarithmic   →  Binary search
O(√n)       →  Square root   →  Primality check (trial division)
O(n)        →  Linear        →  Finding max in unsorted array
O(n log n)  →  Linearithmic  →  Merge sort, Quick sort (average)
O(n²)       →  Quadratic     →  Bubble sort, checking all pairs
O(n³)       →  Cubic         →  Floyd-Warshall, matrix multiplication
O(2ⁿ)       →  Exponential   →  Generating all subsets
O(n!)       →  Factorial     →  Generating all permutations
O(nⁿ)       →  ...           →  Brute force TSP
```

### Visual Growth Comparison

```
Input size →    10      100      1,000     10,000    100,000
─────────────────────────────────────────────────────────────
O(1)            1       1        1         1         1
O(log n)        3       7        10        13        17
O(n)            10      100      1,000     10,000    100,000
O(n log n)      33      664      9,966     132,877   1,660,964
O(n²)           100     10,000   1,000,000 10⁸       10¹⁰
O(2ⁿ)           1,024   10³⁰     ...       ...       ...
```

### How to Calculate Big O

**Rule 1: Drop constants**
```python
# This is O(n), not O(2n)
for i in range(n):    # O(n)
    print(i)
for i in range(n):    # O(n)
    print(i)
# Total: O(n) + O(n) = O(2n) → O(n)
```

**Rule 2: Drop lower-order terms**
```python
# This is O(n²), not O(n² + n)
for i in range(n):           # O(n²)
    for j in range(n):
        print(i, j)
for i in range(n):           # O(n)
    print(i)
# Total: O(n²) + O(n) → O(n²)
```

**Rule 3: Different inputs = different variables**
```python
# This is O(a * b), NOT O(n²)
def print_pairs(list_a, list_b):
    for a in list_a:         # O(a)
        for b in list_b:     # O(b)
            print(a, b)
```

**Rule 4: Logarithms appear when you halve repeatedly**
```python
# This is O(log n) — we divide n by 2 each time
def count_halvings(n):
    count = 0
    while n > 1:
        n = n // 2
        count += 1
    return count
```

---

## 1.2 Big Omega (Ω) — Lower Bound

**What it means:** "My algorithm will take *at least* this much time."

```
Ω(n) means: the algorithm always does at least n operations
```

**Example:** Any comparison-based sorting algorithm is Ω(n log n) — you cannot sort faster than n log n comparisons in the worst case.

```python
# Linear search — Best case is Ω(1) (found at index 0)
# But worst case guarantees at least n comparisons → Ω(n) worst case
def linear_search(arr, target):
    for i, val in enumerate(arr):
        if val == target:
            return i
    return -1
```

---

## 1.3 Big Theta (Θ) — Tight Bound

**What it means:** "My algorithm always takes *exactly* this much time (up to constants)."

Theta means both the upper AND lower bound are the same.

```
If f(n) is O(n²) AND Ω(n²), then f(n) is Θ(n²)
```

**Example:** Merge sort is Θ(n log n) — it always takes n log n time regardless of input.

---

## 1.4 Little o and Little ω

These are **strict** bounds (without equality):

| Notation | Meaning | Analogy |
|----------|---------|---------|
| O(g(n))  | f ≤ c·g (eventually) | ≤ |
| o(g(n))  | f < c·g (eventually, for ALL c) | < |
| Ω(g(n))  | f ≥ c·g (eventually) | ≥ |
| ω(g(n))  | f > c·g (eventually, for ALL c) | > |
| Θ(g(n))  | f = c·g (eventually) | = |

**Example:** n² is o(n³) because n² grows strictly slower than n³.

---

## 1.5 Best, Average, and Worst Case

These are **different from** Big O/Omega/Theta!

| Case | What it measures | Example (Linear Search) |
|------|-----------------|------------------------|
| **Best Case** | Luckiest possible input | Target is first element → O(1) |
| **Average Case** | Expected over random inputs | Target is in middle → O(n/2) → O(n) |
| **Worst Case** | Unluckiest possible input | Target not in array → O(n) |

```python
# Quick Sort complexity depends on the case:
# Best:    O(n log n) — balanced partitions every time
# Average: O(n log n) — random pivots give good splits
# Worst:   O(n²)      — already sorted + always pick first element as pivot
```

---

## 1.6 Space Complexity

Space complexity measures **extra memory** used by an algorithm (beyond the input).

```python
# O(1) space — only uses a few variables
def find_max(arr):
    max_val = arr[0]
    for val in arr:
        if val > max_val:
            max_val = val
    return max_val

# O(n) space — creates new array of same size
def double_array(arr):
    result = []
    for val in arr:
        result.append(val * 2)
    return result

# O(n) space from recursion call stack
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)  # n stack frames
```

### In-Place vs Out-of-Place

| Type | Extra Space | Example |
|------|------------|---------|
| In-place | O(1) | Bubble sort, Insertion sort |
| Out-of-place | O(n) or more | Merge sort, creating new arrays |

---

## 1.7 Amortized Analysis

**What it means:** Some operations are expensive occasionally, but cheap on average over a sequence of operations.

### Example: Dynamic Array (Python list.append)

```python
# Most appends are O(1), but when the array is full,
# it doubles in size — that one operation is O(n)
#
# Sequence: 1, 1, 1, 1, ..., 1, n, 1, 1, ..., 1, 2n, ...
#
# Amortized cost per operation: O(1)

arr = []
for i in range(1000000):
    arr.append(i)  # Amortized O(1) per append
```

**Why it's O(1) amortized:**
- Array of size n → doubles to 2n → copies n elements
- But the next n insertions are all O(1)
- Total cost for n insertions: n + n/2 + n/4 + ... ≈ 2n
- Average per insertion: 2n / n = O(1)

### Three Methods of Amortized Analysis

1. **Aggregate Method:** Total cost of n operations / n
2. **Accounting Method:** Charge each operation extra "credits" to pay for expensive future operations
3. **Potential Method:** Define a potential function Φ; amortized cost = actual cost + ΔΦ

---

## 1.8 Recurrence Relations

Many recursive algorithms have their time complexity expressed as recurrences.

### Common Recurrences and Their Solutions

| Recurrence | Solution | Algorithm |
|-----------|----------|-----------|
| T(n) = T(n/2) + O(1) | O(log n) | Binary Search |
| T(n) = T(n-1) + O(1) | O(n) | Linear traversal |
| T(n) = T(n-1) + O(n) | O(n²) | Selection sort |
| T(n) = 2T(n/2) + O(n) | O(n log n) | Merge Sort |
| T(n) = 2T(n/2) + O(1) | O(n) | Tree traversal |
| T(n) = T(n/2) + O(n) | O(n) | Median finding |
| T(n) = 2T(n-1) + O(1) | O(2ⁿ) | Towers of Hanoi |

### The Master Theorem

For recurrences of the form: **T(n) = aT(n/b) + O(nᵈ)**

Where: a = number of subproblems, b = factor by which input shrinks, d = exponent of work done outside recursion

```
Case 1: If d < log_b(a)  →  T(n) = O(n^(log_b(a)))
Case 2: If d = log_b(a)  →  T(n) = O(nᵈ · log n)
Case 3: If d > log_b(a)  →  T(n) = O(nᵈ)
```

**Example:** Merge Sort: T(n) = 2T(n/2) + O(n)
- a = 2, b = 2, d = 1
- log₂(2) = 1 = d → Case 2
- T(n) = O(n¹ · log n) = O(n log n) ✓

---

## 1.9 P vs NP and Computational Complexity Classes

### Complexity Classes

| Class | Description | Example |
|-------|------------|---------|
| **P** | Solvable in polynomial time | Sorting, shortest path |
| **NP** | Verifiable in polynomial time | Sudoku, SAT |
| **NP-Complete** | Hardest problems in NP | 3-SAT, Traveling Salesman (decision) |
| **NP-Hard** | At least as hard as NP-Complete | Halting problem, TSP optimization |
| **PSPACE** | Solvable with polynomial space | QBF (Quantified Boolean Formulas) |
| **EXPTIME** | Solvable in exponential time | Chess (generalized) |
| **Undecidable** | Cannot be solved by any algorithm | Halting problem |

### The Big Question: P = NP?

```
If P = NP, then every problem whose solution can be
quickly VERIFIED can also be quickly SOLVED.

Most computer scientists believe P ≠ NP, but nobody
has proven it. It's one of the Millennium Prize Problems
(worth $1,000,000).
```

### NP-Complete Problems (Important to Know)

1. **Boolean Satisfiability (SAT)** — Can a boolean formula be satisfied?
2. **3-SAT** — SAT where each clause has exactly 3 literals
3. **Clique** — Does a graph contain a complete subgraph of size k?
4. **Vertex Cover** — Can you cover all edges with ≤ k vertices?
5. **Hamiltonian Cycle** — Is there a cycle visiting every vertex exactly once?
6. **Traveling Salesman (decision)** — Is there a tour of length ≤ k?
7. **Graph Coloring** — Can you color a graph with ≤ k colors?
8. **Subset Sum** — Does a subset add up to a target value?
9. **Knapsack** — Can items fit with maximum value under weight limit?
10. **Independent Set** — Does a graph have an independent set of size k?

### Reductions

**If you can transform Problem A into Problem B in polynomial time, and Problem B is solvable in polynomial time, then so is Problem A.**

```
Problem A ≤_p Problem B  means:
"A is no harder than B"
"If we can solve B, we can solve A"
```

---

## 1.10 Space-Time Tradeoffs

A fundamental principle: you can often trade memory for speed, or vice versa.

| Strategy | Time | Space | Example |
|----------|------|-------|---------|
| Brute force | High | Low | Recompute everything |
| Memoization/Caching | Low | High | Store computed results |
| Lookup tables | O(1) | O(n) | Precompute all answers |
| Compression | Higher processing | Lower storage | Zip files |
| Streaming | Single pass | O(1) or O(k) | Process without storing all data |

```python
# WITHOUT memoization: O(2ⁿ) time, O(n) space (call stack)
def fib_slow(n):
    if n <= 1:
        return n
    return fib_slow(n-1) + fib_slow(n-2)

# WITH memoization: O(n) time, O(n) space (cache + call stack)
def fib_fast(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_fast(n-1, memo) + fib_fast(n-2, memo)
    return memo[n]

# Optimal: O(n) time, O(1) space
def fib_optimal(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

---

## 1.11 Stability in Algorithms

A sorting algorithm is **stable** if elements with equal keys maintain their relative order.

```
Input:  [(A,3), (B,1), (C,3), (D,2)]
Stable sort by number:   [(B,1), (D,2), (A,3), (C,3)]  ← A before C (original order preserved)
Unstable sort by number: [(B,1), (D,2), (C,3), (A,3)]  ← C before A (order changed)
```

**Why it matters:** When sorting by multiple criteria (e.g., sort by last name, then by first name), stability ensures the first sort is preserved.

---

## 1.9 Additional Complexity Concepts

### Smoothed Analysis 🟡

Analyze algorithms on **slightly perturbed** worst-case inputs. Explains why Simplex is fast in practice despite exponential worst case.

```
Worst-case:   adversary picks the absolute worst input → Simplex: exponential
Average-case: random input → Simplex: polynomial
Smoothed:     adversary picks input, then small random perturbation → Simplex: polynomial

Smoothed complexity = max over inputs of E[time on perturbed input]
```

### Parameterized Complexity (FPT) 🟡

Some NP-hard problems become tractable when a **parameter k** is small.

```
Fixed-Parameter Tractable (FPT): O(f(k) · n^c) where c is constant

Example: Vertex Cover
  NP-hard in general, but solvable in O(2^k · n) for cover of size k
  If k = 20, that's ~10⁶ · n — fast even for large n!

FPT ⊆ XP ⊆ W[1] ⊆ W[2] ⊆ ... (W-hierarchy for parameterized hardness)
```

### More Complexity Classes 🟡

| Class | Description | Example |
|-------|-------------|---------|
| **co-NP** | Complement of NP (easy to verify "no") | "Is this formula unsatisfiable?" |
| **BPP** | Solvable by randomized algo with bounded error | Polynomial identity testing |
| **RP** | Randomized poly-time, one-sided error | Miller-Rabin primality |
| **ZPP** | Randomized, always correct, expected poly-time | Randomized quicksort |
| **#P** | Counting solutions (harder than NP decision) | Count SAT solutions |
| **L / NL** | Decidable in log space / nondeterministic log space | Graph reachability (NL) |
| **NC** | Efficiently parallelizable (polylog depth, poly work) | Matrix multiplication |
| **PPAD** | Finding Nash equilibria | 2-player games |

### Information-Theoretic Lower Bounds 🟡

Prove that certain problems require minimum time/operations.

```
Comparison-based sorting: Ω(n log n)
  Proof: decision tree has n! leaves → height ≥ log₂(n!) = Ω(n log n)

Searching in sorted array: Ω(log n)
  Proof: binary decisions on n items → log₂(n) decisions needed

Element distinctness: Ω(n log n) in comparison model
  (Are all elements in an array distinct?)
```

### Polynomial Reductions 🟡

Formal methods for proving a problem is at least as hard as another.

```
Problem A reduces to Problem B (A ≤_p B):
  If we can solve B in poly-time, we can solve A in poly-time.
  Therefore, B is at least as hard as A.

Karp reduction: transform instance of A into instance of B in poly-time
Cook reduction: solve A using poly-many calls to B's oracle

To prove X is NP-hard:
  1. Pick a known NP-hard problem Y
  2. Show Y ≤_p X (reduce Y to X in polynomial time)
```

### Akra-Bazzi Method 🔴

Generalizes Master Theorem to handle non-uniform splits like `T(n) = T(n/3) + T(2n/3) + n`.

```
For T(n) = Σ aᵢ T(bᵢ n) + g(n), find p such that Σ aᵢ bᵢ^p = 1
Then T(n) = Θ(n^p (1 + ∫₁ⁿ g(u)/u^(p+1) du))
```

### Rice's Theorem 🔴

**All non-trivial semantic properties of programs are undecidable.**

You cannot write a program that determines whether an arbitrary program has a particular behavior (halts, produces correct output, etc.). This sets the fundamental boundary of what compilers, linters, and analysis tools can determine.

---

## Key Takeaways

1. **Always analyze complexity** before choosing an algorithm
2. **Big O is about growth rate**, not exact speed
3. **Constants matter in practice** — O(n) with constant 1000 can be slower than O(n²) for small n
4. **Amortized ≠ Average** — amortized is guaranteed over sequences; average is probabilistic
5. **Space matters** — an O(n log n) time algorithm using O(n²) space may be impractical
6. **Know your problem's class** — if it's NP-complete, look for approximations, not exact solutions
7. **Parameterized complexity** — NP-hard doesn't always mean intractable; look for small parameters

---

[← Back to Index](README.md) | [Next: Linear Data Structures →](02-data-structures-linear.md)
