# Chapter 8: Dynamic Programming 🟢🟡🔴

## What is Dynamic Programming?

Dynamic Programming (DP) solves problems by:
1. Breaking them into **overlapping subproblems**
2. Solving each subproblem **only once**
3. Storing results for reuse (**memoization** or **tabulation**)

```
                    fib(5)
                   /      \
              fib(4)      fib(3)
             /    \       /    \
         fib(3)  fib(2) fib(2) fib(1)
         /   \
     fib(2) fib(1)

Without DP: fib(3) computed 2 times, fib(2) computed 3 times → O(2ⁿ)
With DP: each computed once → O(n)
```

### Two Approaches

| Approach | Direction | Method | Pros |
|----------|-----------|--------|------|
| **Top-Down (Memoization)** | Start from big problem | Recursion + cache | Intuitive, only solves needed subproblems |
| **Bottom-Up (Tabulation)** | Start from small problems | Iteration + table | No recursion overhead, can optimize space |

---

## 8.1 Classic DP Problems

### Fibonacci Numbers 🟢

```python
# Top-Down (Memoization)
def fib_memo(n, memo={}):
    if n in memo: return memo[n]
    if n <= 1: return n
    memo[n] = fib_memo(n - 1) + fib_memo(n - 2)
    return memo[n]

# Bottom-Up (Tabulation)
def fib_tab(n):
    if n <= 1: return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# Space-Optimized O(1)
def fib_opt(n):
    if n <= 1: return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

### Climbing Stairs 🟢

How many ways to climb n stairs if you can take 1 or 2 steps?

```python
def climb_stairs(n):
    if n <= 2: return n
    a, b = 1, 2
    for _ in range(3, n + 1):
        a, b = b, a + b
    return b
# Same as Fibonacci! climb(n) = climb(n-1) + climb(n-2)
```

---

## 8.2 Knapsack Problems 🟢

### 0/1 Knapsack

Given items with weights and values, maximize value within weight capacity. Each item used at most once.

```python
def knapsack_01(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            dp[i][w] = dp[i-1][w]  # Don't take item i
            if weights[i-1] <= w:
                dp[i][w] = max(dp[i][w],
                              dp[i-1][w - weights[i-1]] + values[i-1])  # Take it

    return dp[n][capacity]

# Space-optimized 1D
def knapsack_01_optimized(weights, values, capacity):
    dp = [0] * (capacity + 1)
    for i in range(len(weights)):
        for w in range(capacity, weights[i] - 1, -1):  # Reverse order!
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
    return dp[capacity]

# Example
weights = [2, 3, 4, 5]
values = [3, 4, 5, 6]
print(knapsack_01(weights, values, 8))  # 10 (items with weight 3+5)
```

### Unbounded Knapsack

Each item can be used multiple times.

```python
def knapsack_unbounded(weights, values, capacity):
    dp = [0] * (capacity + 1)
    for w in range(1, capacity + 1):
        for i in range(len(weights)):
            if weights[i] <= w:
                dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
    return dp[capacity]
```

### Fractional Knapsack

Can take fractions of items → Greedy (not DP). Take by best value/weight ratio.

---

## 8.3 Longest Common Subsequence (LCS) 🟢

Find the longest subsequence common to two strings.

```
"ABCBDAB" and "BDCAB" → LCS = "BCAB" (length 4)
```

```python
def lcs(s1, s2):
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    # Reconstruct the LCS
    result = []
    i, j = m, n
    while i > 0 and j > 0:
        if s1[i-1] == s2[j-1]:
            result.append(s1[i-1])
            i -= 1
            j -= 1
        elif dp[i-1][j] > dp[i][j-1]:
            i -= 1
        else:
            j -= 1

    return "".join(reversed(result))

print(lcs("ABCBDAB", "BDCAB"))  # "BCAB"
```

**Time:** O(mn) | **Space:** O(mn), can be optimized to O(min(m,n))

---

## 8.4 Longest Increasing Subsequence (LIS) 🟢

Find the longest subsequence where each element is greater than the previous.

```
[10, 9, 2, 5, 3, 7, 101, 18] → LIS = [2, 3, 7, 101] or [2, 5, 7, 18], length 4
```

```python
# O(n²) DP
def lis_dp(arr):
    n = len(arr)
    dp = [1] * n
    for i in range(1, n):
        for j in range(i):
            if arr[j] < arr[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)

# O(n log n) with binary search (patience sorting)
from bisect import bisect_left

def lis_fast(arr):
    tails = []  # tails[i] = smallest tail of all increasing subsequences of length i+1
    for num in arr:
        pos = bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

---

## 8.5 Edit Distance (Levenshtein Distance) 🟢

Minimum operations (insert, delete, replace) to transform one string to another.

```
"kitten" → "sitting"
  kitten → sitten (replace k→s)
  sitten → sittin (replace e→i)
  sittin → sitting (insert g)
  Distance = 3
```

```python
def edit_distance(s1, s2):
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1): dp[i][0] = i
    for j in range(n + 1): dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(
                    dp[i-1][j],     # Delete
                    dp[i][j-1],     # Insert
                    dp[i-1][j-1]    # Replace
                )

    return dp[m][n]
```

---

## 8.6 Coin Change 🟢

Minimum coins to make a given amount.

```python
def coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0

    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i and dp[i - coin] + 1 < dp[i]:
                dp[i] = dp[i - coin] + 1

    return dp[amount] if dp[amount] != float('inf') else -1

# Count number of ways to make change
def coin_change_ways(coins, amount):
    dp = [0] * (amount + 1)
    dp[0] = 1
    for coin in coins:
        for i in range(coin, amount + 1):
            dp[i] += dp[i - coin]
    return dp[amount]
```

---

## 8.7 Matrix Chain Multiplication 🟡

Find optimal parenthesization to minimize scalar multiplications.

```
Matrices: A(10×30) × B(30×5) × C(5×60)
  (A×B)×C = 10×30×5 + 10×5×60 = 1500 + 3000 = 4500
  A×(B×C) = 30×5×60 + 10×30×60 = 9000 + 18000 = 27000

Optimal: (A×B)×C with 4500 multiplications
```

```python
def matrix_chain(dims):
    """dims[i] = rows of matrix i, dims[i+1] = cols of matrix i"""
    n = len(dims) - 1
    dp = [[0] * n for _ in range(n)]

    for length in range(2, n + 1):  # Chain length
        for i in range(n - length + 1):
            j = i + length - 1
            dp[i][j] = float('inf')
            for k in range(i, j):
                cost = dp[i][k] + dp[k+1][j] + dims[i] * dims[k+1] * dims[j+1]
                dp[i][j] = min(dp[i][j], cost)

    return dp[0][n-1]

print(matrix_chain([10, 30, 5, 60]))  # 4500
```

---

## 8.8 Rod Cutting 🟢

Given a rod of length n and prices for each length, find maximum revenue from cutting.

```python
def rod_cutting(prices, n):
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        for j in range(1, i + 1):
            if j <= len(prices):
                dp[i] = max(dp[i], prices[j-1] + dp[i-j])
    return dp[n]
```

---

## 8.9 Egg Drop Problem 🟡

Given k eggs and n floors, find the minimum number of trials needed to determine the critical floor.

```python
# O(kn²) DP
def egg_drop(k, n):
    dp = [[0] * (n + 1) for _ in range(k + 1)]

    for i in range(1, k + 1):
        dp[i][0] = 0
        dp[i][1] = 1
    for j in range(1, n + 1):
        dp[1][j] = j

    for i in range(2, k + 1):
        for j in range(2, n + 1):
            dp[i][j] = float('inf')
            for x in range(1, j + 1):
                cost = 1 + max(dp[i-1][x-1], dp[i][j-x])
                dp[i][j] = min(dp[i][j], cost)

    return dp[k][n]

# Optimized O(kn log n) using binary search
def egg_drop_fast(k, n):
    dp = [[0] * (n + 1) for _ in range(k + 1)]
    for j in range(1, n + 1):
        dp[1][j] = j

    for i in range(2, k + 1):
        for j in range(1, n + 1):
            lo, hi = 1, j
            while lo < hi:
                mid = (lo + hi) // 2
                below = dp[i-1][mid-1]  # Egg breaks
                above = dp[i][j-mid]     # Egg survives
                if below < above:
                    lo = mid + 1
                else:
                    hi = mid
            dp[i][j] = 1 + max(dp[i-1][lo-1], dp[i][j-lo])

    return dp[k][n]
```

---

## 8.10 Palindrome Problems 🟡

### Longest Palindromic Subsequence

```python
def longest_palindrome_subseq(s):
    n = len(s)
    dp = [[0] * n for _ in range(n)]

    for i in range(n):
        dp[i][i] = 1

    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1
            if s[i] == s[j]:
                dp[i][j] = dp[i+1][j-1] + 2
            else:
                dp[i][j] = max(dp[i+1][j], dp[i][j-1])

    return dp[0][n-1]
```

### Palindrome Partitioning (Min Cuts)

```python
def min_palindrome_cuts(s):
    n = len(s)
    # is_pal[i][j] = True if s[i..j] is palindrome
    is_pal = [[False] * n for _ in range(n)]
    for i in range(n): is_pal[i][i] = True
    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1
            is_pal[i][j] = s[i] == s[j] and (length <= 3 or is_pal[i+1][j-1])

    # dp[i] = min cuts for s[0..i]
    dp = list(range(n))
    for i in range(1, n):
        if is_pal[0][i]:
            dp[i] = 0
        else:
            for j in range(1, i + 1):
                if is_pal[j][i]:
                    dp[i] = min(dp[i], dp[j-1] + 1)

    return dp[n-1]
```

---

## 8.11 Interval DP 🟡

DP on intervals/ranges. Solve smaller intervals first, combine for larger ones.

### Burst Balloons

```python
def max_coins(nums):
    nums = [1] + nums + [1]
    n = len(nums)
    dp = [[0] * n for _ in range(n)]

    for length in range(2, n):
        for left in range(n - length):
            right = left + length
            for k in range(left + 1, right):
                dp[left][right] = max(
                    dp[left][right],
                    dp[left][k] + dp[k][right] + nums[left] * nums[k] * nums[right]
                )

    return dp[0][n-1]
```

---

## 8.12 Bitmask DP 🟡

Use bitmask to represent subsets. `mask = 0b1010` means elements at positions 1 and 3 are included.

```python
# TSP with bitmask DP (covered in graph chapter)
def tsp(dist, n):
    dp = [[float('inf')] * n for _ in range(1 << n)]
    dp[1][0] = 0  # Start at city 0

    for mask in range(1 << n):
        for u in range(n):
            if dp[mask][u] == float('inf'):
                continue
            for v in range(n):
                if mask & (1 << v):
                    continue
                new_mask = mask | (1 << v)
                dp[new_mask][v] = min(dp[new_mask][v], dp[mask][u] + dist[u][v])

    full = (1 << n) - 1
    return min(dp[full][u] + dist[u][0] for u in range(n))

# Assignment Problem — assign n workers to n tasks, minimize cost
def min_cost_assignment(cost):
    n = len(cost)
    dp = [float('inf')] * (1 << n)
    dp[0] = 0

    for mask in range(1 << n):
        worker = bin(mask).count('1')
        if worker >= n:
            continue
        for task in range(n):
            if mask & (1 << task):
                continue
            dp[mask | (1 << task)] = min(
                dp[mask | (1 << task)],
                dp[mask] + cost[worker][task]
            )

    return dp[(1 << n) - 1]
```

---

## 8.13 Digit DP 🟡

Count numbers in range [L, R] satisfying some property, digit by digit.

```python
# Count numbers from 1 to n with no two adjacent digits being the same
from functools import lru_cache

def count_no_adjacent_same(n):
    digits = [int(d) for d in str(n)]

    @lru_cache(maxsize=None)
    def dp(pos, prev_digit, tight, started):
        if pos == len(digits):
            return 1 if started else 0

        limit = digits[pos] if tight else 9
        count = 0

        for d in range(0, limit + 1):
            if started and d == prev_digit:
                continue  # Skip if same as previous

            new_tight = tight and (d == limit)
            new_started = started or (d != 0)
            count += dp(pos + 1, d, new_tight, new_started)

        return count

    return dp(0, -1, True, False)
```

---

## 8.14 Tree DP 🟡

DP on tree structures, typically computed bottom-up (from leaves to root).

```python
# Maximum independent set on a tree
def max_independent_set(adj, n, root=0):
    dp = [[0, 0] for _ in range(n)]  # [exclude, include]
    visited = [False] * n

    def dfs(u):
        visited[u] = True
        dp[u][1] = 1  # Include this node
        for v in adj[u]:
            if not visited[v]:
                dfs(v)
                dp[u][0] += max(dp[v][0], dp[v][1])
                dp[u][1] += dp[v][0]

    dfs(root)
    return max(dp[root])

# Diameter of tree via DP
def tree_diameter(adj, n, root=0):
    max_diameter = [0]
    visited = [False] * n

    def dfs(u):
        visited[u] = True
        depths = [0]  # Collect depths of children
        for v in adj[u]:
            if not visited[v]:
                d = dfs(v) + 1
                depths.append(d)

        depths.sort(reverse=True)
        max_diameter[0] = max(max_diameter[0],
                             depths[0] + (depths[1] if len(depths) > 1 else 0))
        return depths[0]

    dfs(root)
    return max_diameter[0]

# Rerooting technique — compute answer for all possible roots
def reroot(adj, n):
    """Count subtree sizes for each vertex as root"""
    size = [1] * n  # Subtree size when 0 is root

    # First DFS: compute for root = 0
    def dfs1(u, parent):
        for v in adj[u]:
            if v != parent:
                dfs1(v, u)
                size[u] += size[v]

    # Second DFS: reroot to compute for all vertices
    answer = [0] * n
    answer[0] = size[0]

    def dfs2(u, parent):
        for v in adj[u]:
            if v != parent:
                # When rerooting from u to v:
                # v gains the "rest of the tree" that was u's other children
                answer[v] = answer[u]  # Simplified; actual logic depends on problem
                dfs2(v, u)

    dfs1(0, -1)
    dfs2(0, -1)
    return answer
```

---

## 8.15 DP Optimization Techniques 🔴

### Convex Hull Trick

Optimize DP recurrences of the form: `dp[i] = min(dp[j] + b[j] * a[i])` where slopes are monotone.

Reduces O(n²) to O(n).

```python
# When slopes are decreasing and queries are increasing
class ConvexHullTrick:
    def __init__(self):
        self.lines = []  # [(slope, intercept)]

    def add_line(self, m, b):
        """Add line y = mx + b"""
        while len(self.lines) >= 2:
            m1, b1 = self.lines[-2]
            m2, b2 = self.lines[-1]
            # Check if last line is unnecessary
            if (b - b1) * (m1 - m2) <= (b2 - b1) * (m1 - m):
                self.lines.pop()
            else:
                break
        self.lines.append((m, b))

    def query(self, x):
        """Get minimum y = mx + b for given x"""
        lo, hi = 0, len(self.lines) - 1
        while lo < hi:
            mid = (lo + hi) // 2
            m1, b1 = self.lines[mid]
            m2, b2 = self.lines[mid + 1]
            if m1 * x + b1 > m2 * x + b2:
                lo = mid + 1
            else:
                hi = mid
        m, b = self.lines[lo]
        return m * x + b
```

### Divide and Conquer Optimization

For DP where `dp[i][j] = min over k of (dp[i-1][k] + C(k,j))` and the optimal k is monotone.

Reduces O(kn²) to O(kn log n).

### Knuth's Optimization

For interval DP where the optimal split point is monotone: if `opt[i][j-1] ≤ opt[i][j] ≤ opt[i+1][j]`, reduces O(n³) to O(n²).

### Aliens Trick (Lambda Optimization / WQS Binary Search)

Transform "solve with exactly k items" into "solve without k constraint but with a penalty λ per item". Binary search on λ.

---

## 8.16 Profile DP (Broken Profile) 🔴

DP over the "profile" of filled cells at the boundary between processed and unprocessed regions. Used for tiling problems.

```python
# Count ways to tile a 3×n grid with 2×1 dominoes
def domino_tiling_3xn(n):
    if n % 2 == 1:
        return 0
    # States represent profile of the boundary
    # This is a simplified version; actual implementation uses bitmask states
    dp = [0] * (n + 1)
    dp[0] = 1
    dp[2] = 3
    for i in range(4, n + 1, 2):
        dp[i] = 4 * dp[i-2] - dp[i-4]
    return dp[n]
```

---

## 8.17 SOS DP (Sum Over Subsets) 🔴

Efficiently compute sum over all subsets of each bitmask.

```python
def sos_dp(f, n):
    """For each mask, compute sum of f[submask] for all submasks of mask"""
    dp = f[:]
    for bit in range(n):
        for mask in range(1 << n):
            if mask & (1 << bit):
                dp[mask] += dp[mask ^ (1 << bit)]
    return dp
```

**Time:** O(n × 2ⁿ) instead of O(3ⁿ) for brute force.

---

## 8.18 DP on DAGs 🟡

Any DP on a DAG can be computed using topological order.

```python
# Longest path in DAG
def longest_path_dag(graph, n):
    from collections import deque

    in_degree = [0] * n
    for u in range(n):
        for v in graph[u]:
            in_degree[v] += 1

    dist = [0] * n
    queue = deque([u for u in range(n) if in_degree[u] == 0])

    while queue:
        u = queue.popleft()
        for v in graph[u]:
            dist[v] = max(dist[v], dist[u] + 1)
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)

    return max(dist)
```

---

## 8.19 Common DP Patterns Summary

| Pattern | Key Idea | Example |
|---------|----------|---------|
| **Linear** | dp[i] depends on dp[i-1], dp[i-2], ... | Fibonacci, climbing stairs |
| **Grid** | dp[i][j] depends on adjacent cells | Unique paths, min path sum |
| **String** | dp[i][j] for two strings | LCS, edit distance |
| **Partition** | Divide into groups | Palindrome partitioning |
| **Interval** | dp[i][j] for subarray [i..j] | Matrix chain, burst balloons |
| **Knapsack** | Select items with constraints | 0/1 knapsack, subset sum |
| **Bitmask** | States as bitmasks | TSP, assignment |
| **Tree** | Bottom-up on tree | MIS on tree, tree diameter |
| **Digit** | Process digits of number | Count numbers with property |
| **Game** | dp[state] = can player win? | Nim, Stone game |
| **Probability** | Expected value recurrences | Dice games, random walks |
| **Monotone Queue** | Sliding window optimization | Jump game, paint houses |
| **Convex Hull Trick** | Li Chao / CHT | Linear cost functions |

### Steps to Solve Any DP Problem

1. **Identify the state** — What information do I need to make a decision?
2. **Define the recurrence** — How does current state relate to smaller states?
3. **Identify base cases** — What are the trivial states?
4. **Determine order** — Bottom-up: smallest to largest. Top-down: just recurse.
5. **Optimize space** — Can I use rolling array? Do I only need previous row?

---

## Additional DP Techniques

### Probability / Expected Value DP 🟡

```python
def expected_dice_rolls(target):
    """Expected number of rolls to reach exactly 'target' with a fair 6-sided die"""
    # dp[i] = expected rolls to go from sum=i to sum≥target
    dp = [0.0] * (target + 7)
    for i in range(target - 1, -1, -1):
        dp[i] = 1 + sum(dp[i + face] for face in range(1, 7)) / 6.0
    return dp[0]

def coupon_collector(n):
    """Expected attempts to collect all n distinct coupons"""
    # After collecting k coupons, prob of new one = (n-k)/n
    # E[next new] = n/(n-k)
    return sum(n / (n - k) for k in range(n))

# Random walk absorption probability
def random_walk_absorption(n, p=0.5):
    """Prob of reaching n before 0, starting at position i, step +1 with prob p"""
    if p == 0.5:
        return [i / n for i in range(n + 1)]
    q = 1 - p
    r = q / p
    return [(1 - r**i) / (1 - r**n) for i in range(n + 1)]
```

### Sprague-Grundy Game Theory DP 🟡

```python
def sprague_grundy(positions, moves_func, max_pos):
    """
    Compute Grundy numbers for combinatorial game.
    Grundy number = 0 means losing position (for player to move).

    moves_func(pos): returns list of positions reachable from pos
    """
    grundy = [0] * (max_pos + 1)
    for pos in range(max_pos + 1):
        reachable = set()
        for next_pos in moves_func(pos):
            if 0 <= next_pos <= max_pos:
                reachable.add(grundy[next_pos])
        # Grundy number = mex (minimum excludant) of reachable Grundy values
        mex = 0
        while mex in reachable:
            mex += 1
        grundy[pos] = mex
    return grundy

# Nim game: XOR of all pile Grundy numbers
def nim_winner(piles):
    """Return True if first player wins in Nim"""
    xor = 0
    for p in piles:
        xor ^= p
    return xor != 0

# Example: Subtraction game — can remove 1, 2, or 3 stones
grundy = sprague_grundy(None, lambda pos: [pos-1, pos-2, pos-3], 20)
# grundy[n] = n % 4 (pattern: 0,1,2,3,0,1,2,3,...)
```

### Convex Hull Trick / Li Chao Tree 🔴

Optimize DP recurrences of the form: `dp[i] = min(dp[j] + b[j]*a[i])` — O(n²) → O(n log n).

```python
class LiChaoTree:
    """Li Chao Segment Tree for min of linear functions y = mx + b"""
    def __init__(self, lo, hi):
        self.lo = lo
        self.hi = hi
        self.line = None  # (m, b) representing y = mx + b
        self.left = None
        self.right = None

    def _eval(self, line, x):
        return line[0] * x + line[1] if line else float('inf')

    def add_line(self, m, b):
        new_line = (m, b)
        self._add(new_line, self.lo, self.hi)

    def _add(self, new_line, lo, hi):
        if self.line is None:
            self.line = new_line
            return

        mid = (lo + hi) // 2
        left_better = self._eval(new_line, lo) < self._eval(self.line, lo)
        mid_better = self._eval(new_line, mid) < self._eval(self.line, mid)

        if mid_better:
            self.line, new_line = new_line, self.line

        if lo == hi:
            return

        if left_better != mid_better:
            if self.left is None:
                self.left = LiChaoTree(lo, mid)
            self.left._add(new_line, lo, mid)
        else:
            if self.right is None:
                self.right = LiChaoTree(mid + 1, hi)
            self.right._add(new_line, mid + 1, hi)

    def query(self, x):
        """Get minimum y value at x"""
        result = self._eval(self.line, x)
        mid = (self.lo + self.hi) // 2
        if x <= mid and self.left:
            result = min(result, self.left.query(x))
        elif x > mid and self.right:
            result = min(result, self.right.query(x))
        return result

# Example: dp[i] = min(dp[j] + cost[j] * weight[i])
# Add line y = cost[j]*x + dp[j], query at x = weight[i]
```

### Monotone Queue Optimization 🟡

For recurrences: `dp[i] = min(dp[j] + cost(j,i))` where j ∈ [i-k, i-1].

```python
from collections import deque

def sliding_window_min_dp(arr, k):
    """dp[i] = min(dp[j] for j in [i-k, i-1]) + arr[i]"""
    n = len(arr)
    dp = [0] * n
    dp[0] = arr[0]
    dq = deque([0])  # Stores indices, dp values are monotonically increasing

    for i in range(1, n):
        # Remove out-of-window elements
        while dq and dq[0] < i - k:
            dq.popleft()

        dp[i] = dp[dq[0]] + arr[i]

        # Maintain monotone queue
        while dq and dp[i] <= dp[dq[-1]]:
            dq.pop()
        dq.append(i)

    return dp

def max_sum_subarray_no_longer_than_k(arr, k):
    """Max sum of subarray with length ≤ k using monotone deque on prefix sums"""
    n = len(arr)
    prefix = [0] * (n + 1)
    for i in range(n):
        prefix[i+1] = prefix[i] + arr[i]

    dq = deque([0])
    result = float('-inf')
    for i in range(1, n + 1):
        while dq and dq[0] < i - k:
            dq.popleft()
        result = max(result, prefix[i] - prefix[dq[0]])
        while dq and prefix[i] <= prefix[dq[-1]]:
            dq.pop()
        dq.append(i)
    return result
```

### Hirschberg's Algorithm 🟡

Compute edit distance / LCS alignment in O(n·m) time but only O(min(n,m)) space.

```python
def hirschberg_lcs(X, Y):
    """Space-efficient LCS using Hirschberg's divide-and-conquer"""
    def lcs_length_row(A, B):
        """Compute last row of LCS DP table in O(len(B)) space"""
        m = len(B)
        prev = [0] * (m + 1)
        curr = [0] * (m + 1)
        for a in A:
            for j in range(1, m + 1):
                if a == B[j-1]:
                    curr[j] = prev[j-1] + 1
                else:
                    curr[j] = max(prev[j], curr[j-1])
            prev, curr = curr, [0] * (m + 1)
        return prev

    def solve(X, Y):
        n, m = len(X), len(Y)
        if n == 0:
            return ""
        if m == 0:
            return ""
        if n == 1:
            return X[0] if X[0] in Y else ""

        mid = n // 2

        # Score from top
        top = lcs_length_row(X[:mid], Y)
        # Score from bottom (reversed)
        bottom = lcs_length_row(X[mid:][::-1], Y[::-1])

        # Find optimal split point
        best_k = 0
        best_score = 0
        for k in range(m + 1):
            score = top[k] + bottom[m - k]
            if score > best_score:
                best_score = score
                best_k = k

        return solve(X[:mid], Y[:best_k]) + solve(X[mid:], Y[best_k:])

    return solve(X, Y)

# Space: O(min(n,m)) vs standard O(n*m)
print(hirschberg_lcs("ABCBDAB", "BDCAB"))  # "BCAB"
```

### Steiner Tree DP 🔴

Find minimum cost tree connecting a set of terminal vertices in a graph.

```python
def steiner_tree(n, edges, terminals):
    """
    Steiner tree DP: O(3^k * n + 2^k * n^2) where k = |terminals|

    dp[S][v] = min cost to connect terminal set S with v as root
    S is a bitmask over terminals
    """
    import heapq

    k = len(terminals)
    INF = float('inf')

    # Build adjacency list
    adj = [[] for _ in range(n)]
    for u, v, w in edges:
        adj[u].append((v, w))
        adj[v].append((u, w))

    # dp[mask][v] = min cost Steiner tree for terminals in mask, rooted at v
    dp = [[INF] * n for _ in range(1 << k)]

    # Base: single terminal
    for i, t in enumerate(terminals):
        dp[1 << i][t] = 0

    for mask in range(1, 1 << k):
        # Combine sub-masks
        sub = (mask - 1) & mask
        while sub > 0:
            comp = mask ^ sub
            if sub < comp:  # Avoid double counting
                sub = (sub - 1) & mask
                continue
            for v in range(n):
                if dp[sub][v] < INF and dp[comp][v] < INF:
                    dp[mask][v] = min(dp[mask][v], dp[sub][v] + dp[comp][v])
            sub = (sub - 1) & mask

        # Dijkstra relaxation
        pq = [(dp[mask][v], v) for v in range(n) if dp[mask][v] < INF]
        heapq.heapify(pq)
        while pq:
            d, u = heapq.heappop(pq)
            if d > dp[mask][u]:
                continue
            for v, w in adj[u]:
                if d + w < dp[mask][v]:
                    dp[mask][v] = d + w
                    heapq.heappush(pq, (dp[mask][v], v))

    full_mask = (1 << k) - 1
    return min(dp[full_mask])
```

---

## 8.20 Common Interview DP Problems 🟢

### House Robber (with Circular Variant)

```python
def rob(nums):
    """Max money without robbing adjacent houses."""
    if len(nums) <= 2:
        return max(nums) if nums else 0
    prev2, prev1 = 0, 0
    for n in nums:
        prev2, prev1 = prev1, max(prev1, prev2 + n)
    return prev1

def rob_circular(nums):
    """Houses in a circle: first and last are adjacent."""
    if len(nums) == 1:
        return nums[0]
    return max(rob(nums[1:]), rob(nums[:-1]))
```

### Best Time to Buy and Sell Stock

```python
# I: One transaction
def max_profit_1(prices):
    min_price, max_profit = float('inf'), 0
    for p in prices:
        min_price = min(min_price, p)
        max_profit = max(max_profit, p - min_price)
    return max_profit

# II: Unlimited transactions
def max_profit_2(prices):
    return sum(max(0, prices[i] - prices[i-1]) for i in range(1, len(prices)))

# III: At most 2 transactions
def max_profit_3(prices):
    buy1 = buy2 = float('inf')
    profit1 = profit2 = 0
    for p in prices:
        buy1 = min(buy1, p)
        profit1 = max(profit1, p - buy1)
        buy2 = min(buy2, p - profit1)  # effective cost
        profit2 = max(profit2, p - buy2)
    return profit2

# IV: At most k transactions
def max_profit_k(k, prices):
    if k >= len(prices) // 2:
        return max_profit_2(prices)
    buy = [float('inf')] * (k + 1)
    profit = [0] * (k + 1)
    for p in prices:
        for i in range(1, k + 1):
            buy[i] = min(buy[i], p - profit[i-1])
            profit[i] = max(profit[i], p - buy[i])
    return profit[k]

# With cooldown
def max_profit_cooldown(prices):
    sold, held, rest = 0, float('-inf'), 0
    for p in prices:
        prev_sold = sold
        sold = held + p
        held = max(held, rest - p)
        rest = max(rest, prev_sold)
    return max(sold, rest)
```

### Word Break

```python
def word_break(s, word_dict):
    """Can s be segmented into dictionary words?"""
    words = set(word_dict)
    dp = [False] * (len(s) + 1)
    dp[0] = True
    for i in range(1, len(s) + 1):
        for j in range(i):
            if dp[j] and s[j:i] in words:
                dp[i] = True
                break
    return dp[-1]
```

### Decode Ways

```python
def num_decodings(s):
    """A=1..Z=26. Count ways to decode digit string."""
    if not s or s[0] == '0':
        return 0
    n = len(s)
    dp = [0] * (n + 1)
    dp[0] = dp[1] = 1
    for i in range(2, n + 1):
        if s[i-1] != '0':
            dp[i] += dp[i-1]
        two_digit = int(s[i-2:i])
        if 10 <= two_digit <= 26:
            dp[i] += dp[i-2]
    return dp[n]
```

### Partition Equal Subset Sum

```python
def can_partition(nums):
    """Can array be split into two subsets with equal sum?"""
    total = sum(nums)
    if total % 2:
        return False
    target = total // 2
    dp = [False] * (target + 1)
    dp[0] = True
    for num in nums:
        for j in range(target, num - 1, -1):
            dp[j] = dp[j] or dp[j - num]
    return dp[target]
```

### Longest Palindromic Subsequence

```python
def lps(s):
    """O(n²) DP. LPS(s) = LCS(s, reverse(s))."""
    n = len(s)
    dp = [[0] * n for _ in range(n)]
    for i in range(n):
        dp[i][i] = 1
    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1
            if s[i] == s[j]:
                dp[i][j] = dp[i+1][j-1] + 2
            else:
                dp[i][j] = max(dp[i+1][j], dp[i][j-1])
    return dp[0][n-1]
```

### Unique Paths (Grid)

```python
def unique_paths(m, n):
    """Count paths from top-left to bottom-right (only right/down)."""
    dp = [1] * n
    for _ in range(1, m):
        for j in range(1, n):
            dp[j] += dp[j-1]
    return dp[-1]

def unique_paths_obstacles(grid):
    """With obstacles (1 = blocked)."""
    n = len(grid[0])
    dp = [0] * n
    dp[0] = 1
    for row in grid:
        for j in range(n):
            if row[j] == 1:
                dp[j] = 0
            elif j > 0:
                dp[j] += dp[j-1]
    return dp[-1]
```

---

[← Previous: Graph Algorithms](07-graph-algorithms.md) | [Next: String Algorithms →](09-string-algorithms.md)
