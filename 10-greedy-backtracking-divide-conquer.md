# Chapter 10: Greedy, Backtracking & Divide and Conquer 🟢🟡🔴

---

# Part A: Greedy Algorithms

## What is Greedy?

Make the **locally optimal choice** at each step, hoping it leads to a globally optimal solution.

```
Greedy works when:
  1. Greedy Choice Property — local optimum leads to global optimum
  2. Optimal Substructure — optimal solution contains optimal sub-solutions

Greedy does NOT always work!
  Counter-example: Coin change with denominations [1, 3, 4], amount=6
    Greedy: 4+1+1 = 3 coins
    Optimal: 3+3 = 2 coins ← DP needed
```

---

## 10.1 Activity Selection 🟢

Select maximum non-overlapping activities.

```python
def activity_selection(activities):
    """activities = [(start, end), ...]"""
    activities.sort(key=lambda x: x[1])  # Sort by end time
    selected = [activities[0]]

    for i in range(1, len(activities)):
        if activities[i][0] >= selected[-1][1]:
            selected.append(activities[i])

    return selected

activities = [(1,4), (3,5), (0,6), (5,7), (3,9), (5,9), (6,10), (8,11), (8,12), (2,14), (12,16)]
print(activity_selection(activities))
# [(1,4), (5,7), (8,11), (12,16)]
```

---

## 10.2 Huffman Coding 🟡

Optimal prefix-free encoding for data compression.

```
Frequencies: a=5, b=9, c=12, d=13, e=16, f=45
Build min-heap, merge two smallest:

        100
       /    \
     45(f)   55
            /    \
          25      30
         /  \    /  \
       12(c) 13(d) 14  16(e)
                  / \
                5(a) 9(b)

Codes: f=0, c=100, d=101, a=1100, b=1101, e=111
```

```python
import heapq

class HuffmanNode:
    def __init__(self, char, freq):
        self.char = char
        self.freq = freq
        self.left = None
        self.right = None

    def __lt__(self, other):
        return self.freq < other.freq

def huffman_coding(freq_map):
    heap = [HuffmanNode(ch, f) for ch, f in freq_map.items()]
    heapq.heapify(heap)

    while len(heap) > 1:
        left = heapq.heappop(heap)
        right = heapq.heappop(heap)
        merged = HuffmanNode(None, left.freq + right.freq)
        merged.left = left
        merged.right = right
        heapq.heappush(heap, merged)

    root = heap[0]
    codes = {}

    def build_codes(node, code):
        if node.char is not None:
            codes[node.char] = code or "0"
            return
        if node.left:  build_codes(node.left, code + "0")
        if node.right: build_codes(node.right, code + "1")

    build_codes(root, "")
    return codes

codes = huffman_coding({'a': 5, 'b': 9, 'c': 12, 'd': 13, 'e': 16, 'f': 45})
for ch, code in sorted(codes.items()):
    print(f"  {ch}: {code}")
```

---

## 10.3 Fractional Knapsack 🟢

```python
def fractional_knapsack(items, capacity):
    """items = [(weight, value), ...]"""
    items.sort(key=lambda x: x[1]/x[0], reverse=True)  # Sort by value/weight

    total_value = 0
    for weight, value in items:
        if capacity >= weight:
            total_value += value
            capacity -= weight
        else:
            total_value += value * (capacity / weight)
            break

    return total_value
```

---

## 10.4 Job Sequencing with Deadlines 🟡

Maximize profit by scheduling jobs within their deadlines.

```python
def job_sequencing(jobs):
    """jobs = [(job_id, deadline, profit), ...]"""
    jobs.sort(key=lambda x: x[2], reverse=True)  # Sort by profit
    max_deadline = max(j[1] for j in jobs)
    slots = [None] * (max_deadline + 1)

    total_profit = 0
    scheduled = []

    for job_id, deadline, profit in jobs:
        for slot in range(deadline, 0, -1):
            if slots[slot] is None:
                slots[slot] = job_id
                total_profit += profit
                scheduled.append(job_id)
                break

    return total_profit, scheduled
```

---

## 10.5 Minimum Platforms 🟢

Minimum platforms needed so no train waits.

```python
def min_platforms(arrivals, departures):
    arrivals.sort()
    departures.sort()

    platforms = 0
    max_platforms = 0
    i = j = 0

    while i < len(arrivals):
        if arrivals[i] <= departures[j]:
            platforms += 1
            max_platforms = max(max_platforms, platforms)
            i += 1
        else:
            platforms -= 1
            j += 1

    return max_platforms
```

---

## 10.6 Interval Scheduling & Merging 🟢

```python
# Merge overlapping intervals
def merge_intervals(intervals):
    intervals.sort()
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], end))
        else:
            merged.append((start, end))
    return merged

# Minimum intervals to remove for non-overlap
def erase_overlap_intervals(intervals):
    intervals.sort(key=lambda x: x[1])
    count = 0
    prev_end = float('-inf')
    for start, end in intervals:
        if start >= prev_end:
            prev_end = end
        else:
            count += 1
    return count
```

---

## 10.7 Other Classic Greedy Problems

### Minimum Spanning Tree → See [Chapter 7](07-graph-algorithms.md)

### Dijkstra's Shortest Path → See [Chapter 7](07-graph-algorithms.md)

### Task Scheduler

```python
from collections import Counter

def least_interval(tasks, n):
    counts = Counter(tasks)
    max_count = max(counts.values())
    max_count_tasks = sum(1 for v in counts.values() if v == max_count)
    return max(len(tasks), (max_count - 1) * (n + 1) + max_count_tasks)
```

### Gas Station (Circular Route)

```python
def can_complete_circuit(gas, cost):
    total_tank = curr_tank = 0
    start = 0
    for i in range(len(gas)):
        diff = gas[i] - cost[i]
        total_tank += diff
        curr_tank += diff
        if curr_tank < 0:
            start = i + 1
            curr_tank = 0
    return start if total_tank >= 0 else -1
```

---

# Part B: Backtracking

## What is Backtracking?

Explore all potential solutions by building candidates incrementally, **abandoning** (backtracking) candidates that cannot lead to valid solutions.

```
                start
              /   |   \
            a     b     c        ← Choose
          / | \
        ab  ac  ...              ← Choose
        |
       abc                       ← Valid? → Add to results
        |
       ab ← backtrack (remove c) ← Undo
```

### Backtracking Template

```python
def backtrack_template(candidates, target):
    """Reusable template: choose → explore → unchoose."""
    result = []

    def backtrack(start, path, remaining):
        # Base case: found a solution
        if remaining == 0:
            result.append(path[:])
            return
        # Pruning: no valid solution possible
        if remaining < 0:
            return

        for i in range(start, len(candidates)):
            # Skip duplicates (if candidates has duplicates)
            if i > start and candidates[i] == candidates[i-1]:
                continue

            # Choose
            path.append(candidates[i])

            # Explore (i+1 for no reuse, i for reuse allowed)
            backtrack(i + 1, path, remaining - candidates[i])

            # Unchoose (backtrack)
            path.pop()

    candidates.sort()  # Sort to enable duplicate skipping
    backtrack(0, [], target)
    return result
```

---

## 10.8 N-Queens 🟡

Place N queens on N×N board so no two attack each other.

```python
def solve_n_queens(n):
    solutions = []
    board = [['.'] * n for _ in range(n)]

    def is_safe(row, col):
        for i in range(row):
            if board[i][col] == 'Q':
                return False
            if col - (row - i) >= 0 and board[i][col-(row-i)] == 'Q':
                return False
            if col + (row - i) < n and board[i][col+(row-i)] == 'Q':
                return False
        return True

    def backtrack(row):
        if row == n:
            solutions.append(["".join(r) for r in board])
            return
        for col in range(n):
            if is_safe(row, col):
                board[row][col] = 'Q'
                backtrack(row + 1)
                board[row][col] = '.'  # Undo

    backtrack(0)
    return solutions

# Optimized with sets for O(1) safety checks
def solve_n_queens_fast(n):
    solutions = []
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col
    queens = []

    def backtrack(row):
        if row == n:
            solutions.append(queens[:])
            return
        for col in range(n):
            if col in cols or (row-col) in diag1 or (row+col) in diag2:
                continue
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)
            queens.append(col)

            backtrack(row + 1)

            cols.discard(col)
            diag1.discard(row - col)
            diag2.discard(row + col)
            queens.pop()

    backtrack(0)
    return solutions
```

---

## 10.9 Sudoku Solver 🟡

```python
def solve_sudoku(board):
    def is_valid(board, row, col, num):
        for i in range(9):
            if board[row][i] == num: return False
            if board[i][col] == num: return False

        box_r, box_c = 3 * (row // 3), 3 * (col // 3)
        for i in range(box_r, box_r + 3):
            for j in range(box_c, box_c + 3):
                if board[i][j] == num:
                    return False
        return True

    def solve():
        for i in range(9):
            for j in range(9):
                if board[i][j] == 0:
                    for num in range(1, 10):
                        if is_valid(board, i, j, num):
                            board[i][j] = num
                            if solve():
                                return True
                            board[i][j] = 0
                    return False
        return True

    solve()
    return board
```

---

## 10.10 Subsets, Permutations, Combinations 🟢

```python
# Generate all subsets
def subsets(nums):
    result = []
    def backtrack(start, current):
        result.append(current[:])
        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()
    backtrack(0, [])
    return result

# Generate all permutations
def permutations(nums):
    result = []
    def backtrack(current, remaining):
        if not remaining:
            result.append(current[:])
            return
        for i in range(len(remaining)):
            current.append(remaining[i])
            backtrack(current, remaining[:i] + remaining[i+1:])
            current.pop()
    backtrack([], nums)
    return result

# Combinations: choose k from n
def combinations(nums, k):
    result = []
    def backtrack(start, current):
        if len(current) == k:
            result.append(current[:])
            return
        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()
    backtrack(0, [])
    return result
```

---

## 10.11 Word Search 🟡

```python
def word_search(board, word):
    rows, cols = len(board), len(board[0])

    def dfs(r, c, idx):
        if idx == len(word):
            return True
        if r < 0 or r >= rows or c < 0 or c >= cols:
            return False
        if board[r][c] != word[idx]:
            return False

        temp = board[r][c]
        board[r][c] = '#'  # Mark visited

        found = (dfs(r+1, c, idx+1) or dfs(r-1, c, idx+1) or
                 dfs(r, c+1, idx+1) or dfs(r, c-1, idx+1))

        board[r][c] = temp  # Restore
        return found

    for r in range(rows):
        for c in range(cols):
            if dfs(r, c, 0):
                return True
    return False
```

---

## 10.12 Graph Coloring 🟡

```python
def graph_coloring(graph, m):
    """Color graph with at most m colors"""
    n = len(graph)
    colors = [0] * n

    def is_safe(node, color):
        return all(colors[neighbor] != color for neighbor in graph[node])

    def backtrack(node):
        if node == n:
            return True
        for color in range(1, m + 1):
            if is_safe(node, color):
                colors[node] = color
                if backtrack(node + 1):
                    return True
                colors[node] = 0
        return False

    if backtrack(0):
        return colors
    return None
```

---

## 10.13 Subset Sum 🟡

```python
def subset_sum(nums, target):
    """Find if any subset sums to target"""
    def backtrack(idx, remaining):
        if remaining == 0:
            return True
        if idx >= len(nums) or remaining < 0:
            return False
        return backtrack(idx + 1, remaining - nums[idx]) or backtrack(idx + 1, remaining)
    return backtrack(0, target)

# Find all subsets that sum to target
def subset_sum_all(nums, target):
    results = []
    def backtrack(idx, current, remaining):
        if remaining == 0:
            results.append(current[:])
            return
        for i in range(idx, len(nums)):
            if nums[i] > remaining:
                break
            current.append(nums[i])
            backtrack(i + 1, current, remaining - nums[i])
            current.pop()
    nums.sort()
    backtrack(0, [], target)
    return results
```

---

## 10.14 Generating Parentheses 🟢

```python
def generate_parentheses(n):
    result = []
    def backtrack(s, open_count, close_count):
        if len(s) == 2 * n:
            result.append(s)
            return
        if open_count < n:
            backtrack(s + '(', open_count + 1, close_count)
        if close_count < open_count:
            backtrack(s + ')', open_count, close_count + 1)
    backtrack("", 0, 0)
    return result
```

---

# Part C: Divide and Conquer

## What is Divide and Conquer?

1. **Divide** the problem into smaller subproblems
2. **Conquer** each subproblem recursively
3. **Combine** the results

```
              Problem
             /       \
        Sub-1         Sub-2
       /    \         /    \
    S-1a   S-1b    S-2a   S-2b
      \     /        \     /
     Merge-1        Merge-2
         \           /
          Final Answer
```

---

## 10.15 Merge Sort 🟢

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

---

## 10.16 Quick Sort 🟢

```python
def quick_sort(arr, lo=0, hi=None):
    if hi is None: hi = len(arr) - 1
    if lo < hi:
        pivot = partition(arr, lo, hi)
        quick_sort(arr, lo, pivot - 1)
        quick_sort(arr, pivot + 1, hi)

def partition(arr, lo, hi):
    pivot = arr[hi]
    i = lo - 1
    for j in range(lo, hi):
        if arr[j] < pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i+1], arr[hi] = arr[hi], arr[i+1]
    return i + 1
```

---

## 10.17 Closest Pair of Points 🟡

```python
import math

def closest_pair(points):
    points.sort()
    return closest_rec(points)

def closest_rec(pts):
    n = len(pts)
    if n <= 3:
        return brute_force_closest(pts)

    mid = n // 2
    mid_x = pts[mid][0]

    dl = closest_rec(pts[:mid])
    dr = closest_rec(pts[mid:])
    d = min(dl, dr)

    # Strip of points within distance d from the dividing line
    strip = [p for p in pts if abs(p[0] - mid_x) < d]
    strip.sort(key=lambda p: p[1])

    # Check strip (at most 7 comparisons per point)
    for i in range(len(strip)):
        j = i + 1
        while j < len(strip) and strip[j][1] - strip[i][1] < d:
            d = min(d, dist(strip[i], strip[j]))
            j += 1

    return d

def dist(p1, p2):
    return math.sqrt((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2)

def brute_force_closest(pts):
    min_d = float('inf')
    for i in range(len(pts)):
        for j in range(i+1, len(pts)):
            min_d = min(min_d, dist(pts[i], pts[j]))
    return min_d
```

**Time:** O(n log n) | **Space:** O(n)

---

## 10.18 Strassen's Matrix Multiplication 🔴

Multiplies two n×n matrices in O(n^2.807) instead of O(n³).

```python
import numpy as np

def strassen(A, B):
    n = len(A)
    if n <= 64:  # Base case: use standard multiplication
        return A @ B

    mid = n // 2
    A11, A12 = A[:mid, :mid], A[:mid, mid:]
    A21, A22 = A[mid:, :mid], A[mid:, mid:]
    B11, B12 = B[:mid, :mid], B[:mid, mid:]
    B21, B22 = B[mid:, :mid], B[mid:, mid:]

    # 7 multiplications instead of 8
    M1 = strassen(A11 + A22, B11 + B22)
    M2 = strassen(A21 + A22, B11)
    M3 = strassen(A11, B12 - B22)
    M4 = strassen(A22, B21 - B11)
    M5 = strassen(A11 + A12, B22)
    M6 = strassen(A21 - A11, B11 + B12)
    M7 = strassen(A12 - A22, B21 + B22)

    C = np.zeros((n, n))
    C[:mid, :mid] = M1 + M4 - M5 + M7
    C[:mid, mid:] = M3 + M5
    C[mid:, :mid] = M2 + M4
    C[mid:, mid:] = M1 - M2 + M3 + M6

    return C
```

---

## 10.19 Karatsuba Multiplication 🟡

Multiply large numbers in O(n^1.585) instead of O(n²).

```python
def karatsuba(x, y):
    if x < 10 or y < 10:
        return x * y

    n = max(len(str(x)), len(str(y)))
    m = n // 2

    high_x, low_x = divmod(x, 10**m)
    high_y, low_y = divmod(y, 10**m)

    z0 = karatsuba(low_x, low_y)
    z2 = karatsuba(high_x, high_y)
    z1 = karatsuba(high_x + low_x, high_y + low_y) - z0 - z2

    return z2 * 10**(2*m) + z1 * 10**m + z0
```

---

## 10.20 Count Inversions 🟡

Count pairs (i, j) where i < j but arr[i] > arr[j].

```python
def count_inversions(arr):
    if len(arr) <= 1:
        return arr, 0

    mid = len(arr) // 2
    left, left_inv = count_inversions(arr[:mid])
    right, right_inv = count_inversions(arr[mid:])

    merged = []
    inversions = left_inv + right_inv
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            merged.append(left[i])
            i += 1
        else:
            merged.append(right[j])
            inversions += len(left) - i  # All remaining left elements form inversions
            j += 1

    merged.extend(left[i:])
    merged.extend(right[j:])
    return merged, inversions

_, inv = count_inversions([2, 4, 1, 3, 5])
print(inv)  # 3: (2,1), (4,1), (4,3)
```

---

## Meet in the Middle 🟡

Split exponential search space in half, solve each half, combine results. Reduces O(2^n) to O(2^{n/2}).

```python
def meet_in_middle_subset_sum(arr, target):
    """
    Check if any subset sums to target.
    Brute force: O(2^n)
    Meet in the middle: O(2^{n/2} log(2^{n/2})) = O(n · 2^{n/2})

    Works well for n ≤ 40 (vs n ≤ 20 for brute force)
    """
    from bisect import bisect_left

    n = len(arr)
    mid = n // 2
    left, right = arr[:mid], arr[mid:]

    # Generate all subset sums for each half
    def all_subset_sums(a):
        sums = [0]
        for x in a:
            sums += [s + x for s in sums]
        return sums

    left_sums = all_subset_sums(left)
    right_sums = sorted(all_subset_sums(right))

    # For each left sum, binary search for complement in right sums
    for ls in left_sums:
        complement = target - ls
        idx = bisect_left(right_sums, complement)
        if idx < len(right_sums) and right_sums[idx] == complement:
            return True
    return False

# Handles n=40 easily (2^20 ≈ 1M per half)
print(meet_in_middle_subset_sum([3,7,1,8,12,5,2,9,4,6,11,15,13,14,10,
                                  17,19,20,16,18,21,23,22,25,24,27,26,
                                  29,28,30,31,33,32,35,34,37,36,38,39,40], 420))

def meet_in_middle_closest_sum(arr, target):
    """
    Find subset sum closest to target — O(n · 2^{n/2})
    """
    n = len(arr)
    mid = n // 2

    def all_subset_sums(a):
        sums = [0]
        for x in a:
            sums += [s + x for s in sums]
        return sums

    left_sums = all_subset_sums(arr[:mid])
    right_sums = sorted(all_subset_sums(arr[mid:]))

    best = float('inf')
    from bisect import bisect_left
    for ls in left_sums:
        complement = target - ls
        idx = bisect_left(right_sums, complement)
        for j in [idx - 1, idx]:
            if 0 <= j < len(right_sums):
                best = min(best, abs(target - (ls + right_sums[j])))

    return target - best  # or target + best, whichever is closer

# Applications:
# - Subset sum for n ≤ 40
# - 4SUM problem (split into two 2SUM halves)
# - Closest pair of sums
# - Knapsack for large n, small values
```

---

## Additional Topics

### Matroid Theory & Greedy Algorithms 🟡

A **matroid** is a combinatorial structure that generalizes the concept of linear independence. If a problem has matroid structure, greedy gives optimal solution.

```
Matroid (E, I):
  E = ground set (elements)
  I = collection of "independent" sets, satisfying:
    1. ∅ ∈ I (empty set is independent)
    2. If A ∈ I and B ⊂ A, then B ∈ I (hereditary)
    3. If A, B ∈ I and |A| < |B|, then ∃ x ∈ B\A such that A ∪ {x} ∈ I (exchange)

Examples of matroids:
  - Graphic matroid: E = edges, I = acyclic edge sets (forests) → MST is greedy!
  - Uniform matroid: I = sets of size ≤ k
  - Partition matroid: elements partitioned, ≤ k_i from each part
  - Transversal matroid: matchable subsets in bipartite graph
```

```python
def greedy_matroid(elements, weights, is_independent):
    """
    Generic greedy algorithm on a weighted matroid.
    Maximizes total weight of maximum independent set.

    Works when (elements, is_independent) forms a matroid.
    """
    # Sort by weight descending
    sorted_elements = sorted(elements, key=lambda e: weights[e], reverse=True)

    result = set()
    for e in sorted_elements:
        candidate = result | {e}
        if is_independent(candidate):
            result = candidate

    return result, sum(weights[e] for e in result)

# Example: Graphic matroid (MST)
# is_independent checks if adding edge creates no cycle (Union-Find)
```

### Matroid Intersection 🔴

Find max-weight common independent set of two matroids. Generalizes bipartite matching.

```
Matroid Intersection Theorem:
  max |common independent set| = min over all A ⊂ E of (r₁(A) + r₂(E\A))
  where r₁, r₂ are rank functions of the two matroids.

Applications:
  - Colorful spanning tree
  - Arborescence packing
  - Bipartite matching (intersection of two partition matroids)

Time: O(r^{1.5} · n) augmenting path algorithm
```

### Weighted Interval Scheduling 🟡

Select maximum-weight subset of non-overlapping intervals.

```python
from bisect import bisect_right

def weighted_interval_scheduling(intervals):
    """
    intervals: [(start, end, weight), ...]
    Returns maximum total weight of non-overlapping intervals.
    O(n log n)
    """
    intervals.sort(key=lambda x: x[1])  # Sort by end time
    n = len(intervals)

    # p[i] = largest index j < i such that interval j doesn't overlap with i
    ends = [iv[1] for iv in intervals]
    def find_last_compatible(i):
        start_i = intervals[i][0]
        idx = bisect_right(ends, start_i, 0, i) - 1
        return idx

    # DP
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        weight = intervals[i-1][2]
        p = find_last_compatible(i-1)
        dp[i] = max(dp[i-1], weight + dp[p + 1])

    return dp[n]

intervals = [(1, 3, 5), (2, 5, 6), (4, 6, 5), (6, 7, 4), (5, 8, 11), (7, 9, 2)]
print(weighted_interval_scheduling(intervals))  # 15: (1,3,5) + (4,6,5) + (6,7,4) or similar
```

### Branch and Bound (General Framework) 🟡

```python
import heapq

def branch_and_bound(initial_state, branch, bound, is_solution, objective):
    """
    Generic branch-and-bound framework.

    branch(state): returns list of child states
    bound(state): returns optimistic bound (upper bound for max, lower for min)
    is_solution(state): returns True if state is a complete solution
    objective(state): returns objective value of complete solution
    """
    best_value = float('-inf')
    best_solution = None

    # Priority queue: (-bound, state) for max problem
    pq = [(-bound(initial_state), initial_state)]
    nodes_explored = 0

    while pq:
        neg_bnd, state = heapq.heappop(pq)
        nodes_explored += 1

        # Prune if bound can't beat best
        if -neg_bnd <= best_value:
            continue

        if is_solution(state):
            val = objective(state)
            if val > best_value:
                best_value = val
                best_solution = state
            continue

        for child in branch(state):
            child_bound = bound(child)
            if child_bound > best_value:  # Only explore promising children
                heapq.heappush(pq, (-child_bound, child))

    return best_solution, best_value, nodes_explored
```

### Constraint Propagation (Arc Consistency) 🟡

Core technique in constraint satisfaction problems (CSPs), used with backtracking.

```python
def ac3(domains, constraints):
    """
    Arc Consistency Algorithm 3 (AC-3)

    domains: {var: set of possible values}
    constraints: list of (var1, var2, constraint_func)

    Returns reduced domains (or None if inconsistent)
    """
    from collections import deque

    # Build queue of all arcs
    queue = deque()
    for x, y, _ in constraints:
        queue.append((x, y))
        queue.append((y, x))

    while queue:
        xi, xj = queue.popleft()
        if revise(domains, xi, xj, constraints):
            if len(domains[xi]) == 0:
                return None  # Inconsistent
            # Add all arcs (xk, xi) where xk is neighbor of xi
            for x, y, _ in constraints:
                if y == xi and x != xj:
                    queue.append((x, xi))
                elif x == xi and y != xj:
                    queue.append((y, xi))
    return domains

def revise(domains, xi, xj, constraints):
    revised = False
    for constraint in constraints:
        x, y, func = constraint
        if (x == xi and y == xj) or (x == xj and y == xi):
            to_remove = set()
            for vi in domains[xi]:
                if not any(func(vi, vj) if x == xi else func(vj, vi)
                          for vj in domains[xj]):
                    to_remove.add(vi)
            if to_remove:
                domains[xi] -= to_remove
                revised = True
    return revised

# Example: Sudoku-like constraints
# domains = {(r,c): {1..9} for each cell}
# constraints = [(cell1, cell2, lambda a,b: a != b) for same row/col/box]
```

### Algorithm X with Dancing Links (DLX) 🔴

Knuth's algorithm for exact cover problems — N-Queens, Sudoku, Pentominoes.

```python
def algorithm_x(matrix, columns=None, solution=None):
    """
    Solve exact cover: find rows that cover each column exactly once.
    matrix: list of rows, each row is a set of column indices.
    """
    if solution is None:
        solution = []
    if columns is None:
        columns = set()
        for row in matrix:
            columns.update(row)

    if not columns:
        return [solution[:]]  # Found solution!

    results = []

    # Choose column with fewest 1s (MRV heuristic)
    min_col = min(columns, key=lambda c: sum(1 for r in matrix if c in r))

    for i, row in enumerate(matrix):
        if min_col not in row:
            continue

        solution.append(i)

        # Cover: remove columns covered by this row and conflicting rows
        new_columns = columns - row
        new_matrix = [r for r in matrix if not (r & row)]

        results.extend(algorithm_x(new_matrix, new_columns, solution))

        solution.pop()

    return results

# Sudoku as exact cover
# Columns: cell filled, row has digit, col has digit, box has digit
# Rows: placing digit d at position (r,c)
```

### Knight's Tour 🟡

Visit every square on a chessboard exactly once using Warnsdorff's heuristic.

```python
def knights_tour(n=8):
    """Find knight's tour on n×n board using Warnsdorff's rule"""
    moves = [(-2,-1),(-2,1),(-1,-2),(-1,2),(1,-2),(1,2),(2,-1),(2,1)]
    board = [[-1]*n for _ in range(n)]

    def degree(x, y):
        """Count available moves from (x,y)"""
        count = 0
        for dx, dy in moves:
            nx, ny = x+dx, y+dy
            if 0 <= nx < n and 0 <= ny < n and board[nx][ny] == -1:
                count += 1
        return count

    def solve(x, y, move_num):
        board[x][y] = move_num
        if move_num == n*n - 1:
            return True

        # Warnsdorff's: try squares with fewest onward moves first
        next_moves = []
        for dx, dy in moves:
            nx, ny = x+dx, y+dy
            if 0 <= nx < n and 0 <= ny < n and board[nx][ny] == -1:
                next_moves.append((degree(nx, ny), nx, ny))
        next_moves.sort()

        for _, nx, ny in next_moves:
            if solve(nx, ny, move_num + 1):
                return True

        board[x][y] = -1
        return False

    solve(0, 0, 0)
    return board
```

---

## 10.21 Paradigm Comparison

| Aspect | Greedy | Backtracking | Divide & Conquer | DP |
|--------|--------|-------------|-----------------|-----|
| **Approach** | Local optimum | Try all, prune | Split, solve, merge | Optimal substructure |
| **Guarantee** | Not always optimal | Finds all/optimal | Correct | Optimal |
| **Overlapping?** | N/A | No | No | Yes |
| **Complexity** | Usually fast | Exponential | O(n log n) typical | Polynomial |
| **Example** | Activity select | N-Queens | Merge sort | Knapsack |
| **When to use** | Greedy choice works | Need all solutions | Independent subprobs | Overlapping subprobs |
| **Matroid** | Guarantees opt | N/A | N/A | N/A |

---

[← Previous: String Algorithms](09-string-algorithms.md) | [Next: Math & Bit Algorithms →](11-mathematical-and-bit-algorithms.md)
