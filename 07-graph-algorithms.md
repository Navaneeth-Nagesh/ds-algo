# Chapter 7: Graph Algorithms 🟢🟡🔴

---

## 7.1 Breadth-First Search (BFS) 🟢

Explore all neighbors at the current depth before moving deeper. Uses a **queue**.

```
       0
      / \
     1   2        BFS order: 0, 1, 2, 3, 4, 5
    / \   \
   3   4   5
```

```python
from collections import deque

def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return order

# BFS shortest path in unweighted graph
def bfs_shortest_path(graph, start, end):
    visited = set([start])
    queue = deque([(start, [start])])

    while queue:
        node, path = queue.popleft()
        if node == end:
            return path
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, path + [neighbor]))

    return None  # No path exists

# BFS level-by-level
def bfs_levels(graph, start):
    visited = set([start])
    queue = deque([start])
    levels = []

    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node)
            for neighbor in graph[node]:
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(neighbor)
        levels.append(level)

    return levels
```

**Time:** O(V + E) | **Space:** O(V)

**Use for:** Shortest path (unweighted), level-order traversal, connected components, bipartite check.

---

## 7.2 Depth-First Search (DFS) 🟢

Explore as deep as possible before backtracking. Uses a **stack** (or recursion).

```
       0
      / \
     1   2        DFS order: 0, 1, 3, 4, 2, 5
    / \   \
   3   4   5
```

```python
# Recursive DFS
def dfs_recursive(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    print(node, end=" ")
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs_recursive(graph, neighbor, visited)

# Iterative DFS
def dfs_iterative(graph, start):
    visited = set()
    stack = [start]
    order = []

    while stack:
        node = stack.pop()
        if node not in visited:
            visited.add(node)
            order.append(node)
            for neighbor in reversed(graph[node]):
                if neighbor not in visited:
                    stack.append(neighbor)

    return order

# DFS to detect cycle in directed graph
def has_cycle_directed(graph, n):
    WHITE, GRAY, BLACK = 0, 1, 2
    color = [WHITE] * n

    def dfs(node):
        color[node] = GRAY  # Being processed
        for neighbor in graph[node]:
            if color[neighbor] == GRAY:
                return True  # Back edge = cycle!
            if color[neighbor] == WHITE and dfs(neighbor):
                return True
        color[node] = BLACK  # Fully processed
        return False

    return any(color[i] == WHITE and dfs(i) for i in range(n))

# DFS to find all paths
def find_all_paths(graph, start, end, path=None):
    if path is None:
        path = []
    path = path + [start]

    if start == end:
        return [path]

    paths = []
    for neighbor in graph[start]:
        if neighbor not in path:
            paths.extend(find_all_paths(graph, neighbor, end, path))
    return paths
```

**Time:** O(V + E) | **Space:** O(V)

**Use for:** Cycle detection, topological sort, connected components, path finding, maze solving.

---

## 7.3 Dijkstra's Algorithm 🟢

Find shortest paths from a source to all other vertices in a **non-negative weighted** graph.

```
        1
    A ────── B
    |        |
  4 |        | 2
    |    1   |
    C ────── D
     \      /
    5 \  / 3
       E

Shortest from A:
  A→A: 0
  A→B: 1
  A→D: 3 (A→B→D)
  A→C: 4
  A→E: 6 (A→B→D→E)
```

```python
import heapq

def dijkstra(graph, start):
    """graph[u] = [(v, weight), ...]"""
    distances = {start: 0}
    heap = [(0, start)]
    parent = {start: None}

    while heap:
        dist, u = heapq.heappop(heap)

        if dist > distances.get(u, float('inf')):
            continue  # Skip outdated entries

        for v, weight in graph[u]:
            new_dist = dist + weight
            if new_dist < distances.get(v, float('inf')):
                distances[v] = new_dist
                parent[v] = u
                heapq.heappush(heap, (new_dist, v))

    return distances, parent

def get_path(parent, target):
    path = []
    while target is not None:
        path.append(target)
        target = parent[target]
    return path[::-1]

# Usage
graph = {
    'A': [('B', 1), ('C', 4)],
    'B': [('A', 1), ('D', 2)],
    'C': [('A', 4), ('D', 1), ('E', 5)],
    'D': [('B', 2), ('C', 1), ('E', 3)],
    'E': [('C', 5), ('D', 3)]
}
distances, parent = dijkstra(graph, 'A')
print(distances)  # {'A': 0, 'B': 1, 'C': 4, 'D': 3, 'E': 6}
print(get_path(parent, 'E'))  # ['A', 'B', 'D', 'E']
```

**Time:** O((V + E) log V) with binary heap | O(V² + E) with array | O(V log V + E) with Fibonacci heap

**Limitation:** Does NOT work with negative weights.

---

## 7.4 Bellman-Ford Algorithm 🟡

Find shortest paths from source to all vertices. **Works with negative weights** and detects negative cycles.

```python
def bellman_ford(edges, n, source):
    """edges = [(u, v, weight), ...]"""
    dist = [float('inf')] * n
    dist[source] = 0
    parent = [-1] * n

    # Relax all edges V-1 times
    for _ in range(n - 1):
        for u, v, w in edges:
            if dist[u] != float('inf') and dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                parent[v] = u

    # Check for negative cycles (V-th iteration)
    for u, v, w in edges:
        if dist[u] != float('inf') and dist[u] + w < dist[v]:
            return None  # Negative cycle detected!

    return dist, parent
```

**Time:** O(V × E) | **Space:** O(V)

---

## 7.5 Floyd-Warshall Algorithm 🟡

Find shortest paths between **ALL pairs** of vertices. Works with negative weights (no negative cycles).

```python
def floyd_warshall(graph_matrix):
    """graph_matrix[i][j] = weight (inf if no edge)"""
    n = len(graph_matrix)
    dist = [row[:] for row in graph_matrix]  # Copy

    # For each intermediate vertex k
    for k in range(n):
        for i in range(n):
            for j in range(n):
                if dist[i][k] + dist[k][j] < dist[i][j]:
                    dist[i][j] = dist[i][k] + dist[k][j]

    return dist

# Detect negative cycle: check if any dist[i][i] < 0
```

**Time:** O(V³) | **Space:** O(V²)

---

## 7.6 A* Search Algorithm 🟡

An informed search that uses a **heuristic** to guide the search toward the goal. Guaranteed optimal if heuristic is admissible (never overestimates).

```
f(n) = g(n) + h(n)
  g(n) = actual cost from start to n
  h(n) = estimated cost from n to goal (heuristic)
  f(n) = estimated total cost through n
```

```python
import heapq

def a_star(graph, start, goal, heuristic):
    """graph[u] = [(v, cost), ...], heuristic(node) → estimated cost to goal"""
    open_set = [(heuristic(start), 0, start, [start])]
    visited = set()

    while open_set:
        f, g, current, path = heapq.heappop(open_set)

        if current == goal:
            return path, g

        if current in visited:
            continue
        visited.add(current)

        for neighbor, cost in graph[current]:
            if neighbor not in visited:
                new_g = g + cost
                new_f = new_g + heuristic(neighbor)
                heapq.heappush(open_set, (new_f, new_g, neighbor, path + [neighbor]))

    return None, float('inf')

# Common heuristics for grid:
def manhattan_distance(a, b):
    return abs(a[0] - b[0]) + abs(a[1] - b[1])

def euclidean_distance(a, b):
    return ((a[0] - b[0])**2 + (a[1] - b[1])**2) ** 0.5
```

**Time:** Depends on heuristic. With perfect heuristic: O(1). Worst case: O(b^d) where b = branching factor, d = depth.

**Used in:** GPS navigation, video games, robotics, puzzle solving.

---

## 7.7 Topological Sort 🟢

Order vertices of a DAG so that for every edge u → v, u appears before v.

```
  5 → 2 → 3 → 1
  5 → 0 → 2
  4 → 0
  4 → 1

Topological orders: [4, 5, 0, 2, 3, 1] or [5, 4, 0, 2, 3, 1] etc.
```

```python
# Method 1: Kahn's Algorithm (BFS-based)
from collections import deque

def topological_sort_kahn(graph, n):
    in_degree = [0] * n
    for u in range(n):
        for v in graph[u]:
            in_degree[v] += 1

    queue = deque([u for u in range(n) if in_degree[u] == 0])
    order = []

    while queue:
        u = queue.popleft()
        order.append(u)
        for v in graph[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)

    if len(order) != n:
        return None  # Cycle detected! (not a DAG)
    return order

# Method 2: DFS-based
def topological_sort_dfs(graph, n):
    visited = [False] * n
    stack = []

    def dfs(u):
        visited[u] = True
        for v in graph[u]:
            if not visited[v]:
                dfs(v)
        stack.append(u)

    for u in range(n):
        if not visited[u]:
            dfs(u)

    return stack[::-1]
```

**Time:** O(V + E) | **Use for:** Build systems, task scheduling, course prerequisites.

---

## 7.8 Kruskal's Algorithm (MST) 🟢

Find Minimum Spanning Tree by greedily adding the cheapest edge that doesn't form a cycle.

```python
def kruskal(edges, n):
    """edges = [(weight, u, v), ...], n = number of vertices"""
    edges.sort()  # Sort by weight
    uf = UnionFind(n)
    mst = []
    total_weight = 0

    for weight, u, v in edges:
        if not uf.connected(u, v):
            uf.union(u, v)
            mst.append((u, v, weight))
            total_weight += weight
            if len(mst) == n - 1:
                break

    return mst, total_weight
```

**Time:** O(E log E) (dominated by sorting)

---

## 7.9 Prim's Algorithm (MST) 🟢

Grow MST from a starting vertex by always adding the cheapest edge connecting the tree to a non-tree vertex.

```python
import heapq

def prim(graph, n, start=0):
    """graph[u] = [(v, weight), ...]"""
    visited = [False] * n
    heap = [(0, start, -1)]  # (weight, vertex, parent)
    mst = []
    total_weight = 0

    while heap and len(mst) < n:
        weight, u, parent = heapq.heappop(heap)
        if visited[u]:
            continue
        visited[u] = True
        total_weight += weight
        if parent != -1:
            mst.append((parent, u, weight))

        for v, w in graph[u]:
            if not visited[v]:
                heapq.heappush(heap, (w, v, u))

    return mst, total_weight
```

**Time:** O(E log V) with binary heap | O(E + V log V) with Fibonacci heap

---

## 7.10 Borůvka's Algorithm (MST) 🟡

Find MST by having each component add its cheapest outgoing edge simultaneously.

```python
def boruvka(edges, n):
    uf = UnionFind(n)
    mst_weight = 0
    num_components = n

    while num_components > 1:
        # Find cheapest outgoing edge for each component
        cheapest = [None] * n
        for w, u, v in edges:
            cu, cv = uf.find(u), uf.find(v)
            if cu != cv:
                if cheapest[cu] is None or w < cheapest[cu][0]:
                    cheapest[cu] = (w, u, v)
                if cheapest[cv] is None or w < cheapest[cv][0]:
                    cheapest[cv] = (w, u, v)

        # Add cheapest edges
        for i in range(n):
            if cheapest[i] is not None:
                w, u, v = cheapest[i]
                if uf.union(u, v):
                    mst_weight += w
                    num_components -= 1

    return mst_weight
```

**Time:** O(E log V) | Good for parallel implementations.

---

## 7.11 Tarjan's Algorithm (Strongly Connected Components) 🟡

Find all SCCs in a directed graph. An SCC is a maximal set of vertices where every vertex is reachable from every other.

```python
def tarjan_scc(graph, n):
    index_counter = [0]
    stack = []
    on_stack = [False] * n
    index = [-1] * n
    lowlink = [-1] * n
    sccs = []

    def strongconnect(v):
        index[v] = lowlink[v] = index_counter[0]
        index_counter[0] += 1
        stack.append(v)
        on_stack[v] = True

        for w in graph[v]:
            if index[w] == -1:
                strongconnect(w)
                lowlink[v] = min(lowlink[v], lowlink[w])
            elif on_stack[w]:
                lowlink[v] = min(lowlink[v], index[w])

        # If v is a root of an SCC
        if lowlink[v] == index[v]:
            scc = []
            while True:
                w = stack.pop()
                on_stack[w] = False
                scc.append(w)
                if w == v:
                    break
            sccs.append(scc)

    for v in range(n):
        if index[v] == -1:
            strongconnect(v)

    return sccs
```

**Time:** O(V + E)

---

## 7.12 Kosaraju's Algorithm (SCC) 🟡

Alternative SCC algorithm using two DFS passes:
1. DFS on original graph → get finish order
2. DFS on reversed graph in reverse finish order → each DFS tree is an SCC

```python
def kosaraju_scc(graph, n):
    # Step 1: Get finish order
    visited = [False] * n
    finish_order = []

    def dfs1(u):
        visited[u] = True
        for v in graph[u]:
            if not visited[v]:
                dfs1(v)
        finish_order.append(u)

    for u in range(n):
        if not visited[u]:
            dfs1(u)

    # Step 2: Build reverse graph
    reverse_graph = [[] for _ in range(n)]
    for u in range(n):
        for v in graph[u]:
            reverse_graph[v].append(u)

    # Step 3: DFS on reverse graph in reverse finish order
    visited = [False] * n
    sccs = []

    def dfs2(u, scc):
        visited[u] = True
        scc.append(u)
        for v in reverse_graph[u]:
            if not visited[v]:
                dfs2(v, scc)

    for u in reversed(finish_order):
        if not visited[u]:
            scc = []
            dfs2(u, scc)
            sccs.append(scc)

    return sccs
```

---

## 7.13 Articulation Points & Bridges 🟡

- **Articulation point:** A vertex whose removal disconnects the graph
- **Bridge:** An edge whose removal disconnects the graph

```python
def find_bridges_and_articulation_points(graph, n):
    disc = [-1] * n
    low = [-1] * n
    parent = [-1] * n
    timer = [0]
    bridges = []
    ap = set()

    def dfs(u):
        disc[u] = low[u] = timer[0]
        timer[0] += 1
        children = 0

        for v in graph[u]:
            if disc[v] == -1:
                children += 1
                parent[v] = u
                dfs(v)
                low[u] = min(low[u], low[v])

                # Articulation point conditions
                if parent[u] == -1 and children > 1:
                    ap.add(u)  # Root with 2+ children
                if parent[u] != -1 and low[v] >= disc[u]:
                    ap.add(u)  # Non-root with no back edge from subtree

                # Bridge condition
                if low[v] > disc[u]:
                    bridges.append((u, v))

            elif v != parent[u]:
                low[u] = min(low[u], disc[v])

    for u in range(n):
        if disc[u] == -1:
            dfs(u)

    return bridges, ap
```

**Time:** O(V + E)

---

## 7.14 Eulerian Path & Circuit 🟡

- **Eulerian Path:** Visits every EDGE exactly once
- **Eulerian Circuit:** Eulerian path that starts and ends at the same vertex

**Existence conditions:**
- Circuit: All vertices have even degree (undirected) or equal in/out-degree (directed)
- Path: Exactly 0 or 2 vertices with odd degree

```python
# Hierholzer's Algorithm — O(E)
def find_eulerian_circuit(graph):
    """graph = adjacency list (modify in place by removing edges)"""
    from collections import defaultdict, deque

    adj = defaultdict(list)
    for u in graph:
        for v in graph[u]:
            adj[u].append(v)

    stack = [0]  # Start from vertex 0
    circuit = deque()

    while stack:
        v = stack[-1]
        if adj[v]:
            u = adj[v].pop()
            stack.append(u)
        else:
            circuit.appendleft(stack.pop())

    return list(circuit)
```

---

## 7.15 Hamiltonian Path & Cycle 🔴

Visit every VERTEX exactly once. This is **NP-complete** — no known polynomial algorithm.

```python
# Backtracking solution — O(n!)
def hamiltonian_path(graph, n):
    path = [0]
    visited = {0}

    def backtrack():
        if len(path) == n:
            return True
        for neighbor in graph[path[-1]]:
            if neighbor not in visited:
                visited.add(neighbor)
                path.append(neighbor)
                if backtrack():
                    return True
                path.pop()
                visited.remove(neighbor)
        return False

    return path if backtrack() else None

# DP with bitmask — O(2ⁿ × n²)
def hamiltonian_path_dp(graph, n):
    """dp[mask][i] = True if there's a path visiting vertices in mask, ending at i"""
    dp = [[False] * n for _ in range(1 << n)]

    for i in range(n):
        dp[1 << i][i] = True

    for mask in range(1 << n):
        for u in range(n):
            if not dp[mask][u]:
                continue
            for v in graph[u]:
                if not (mask & (1 << v)):
                    dp[mask | (1 << v)][v] = True

    full_mask = (1 << n) - 1
    return any(dp[full_mask][i] for i in range(n))
```

---

## 7.16 Johnson's Algorithm 🟡

All-pairs shortest paths. Better than Floyd-Warshall for sparse graphs.

1. Add a new vertex connected to all others with weight 0
2. Run Bellman-Ford from new vertex to get reweighting values
3. Reweight all edges to make them non-negative
4. Run Dijkstra from each vertex

**Time:** O(VE + V² log V) — better than O(V³) for sparse graphs.

---

## 7.17 Graph Coloring 🟡

Assign colors to vertices such that no two adjacent vertices share a color.

```python
# Greedy coloring — O(V + E), not optimal but simple
def greedy_coloring(graph, n):
    colors = [-1] * n

    for u in range(n):
        # Find colors used by neighbors
        used = set()
        for v in graph[u]:
            if colors[v] != -1:
                used.add(colors[v])

        # Assign smallest available color
        color = 0
        while color in used:
            color += 1
        colors[u] = color

    return colors

# Check if graph is bipartite (2-colorable) — O(V + E)
def is_bipartite(graph, n):
    color = [-1] * n

    for start in range(n):
        if color[start] != -1:
            continue
        queue = deque([start])
        color[start] = 0
        while queue:
            u = queue.popleft()
            for v in graph[u]:
                if color[v] == -1:
                    color[v] = 1 - color[u]
                    queue.append(v)
                elif color[v] == color[u]:
                    return False
    return True
```

---

## 7.18 Minimum Spanning Arborescence (Edmonds/Chu-Liu) 🔴

MST for **directed graphs**. Find a minimum-weight rooted spanning tree.

**Time:** O(EV) or O(E log V) with optimizations.

---

## 7.19 Shortest Path Faster Algorithm (SPFA) 🟡

A practical optimization of Bellman-Ford using a queue. Average case much faster.

```python
from collections import deque

def spfa(graph, n, source):
    dist = [float('inf')] * n
    dist[source] = 0
    in_queue = [False] * n
    queue = deque([source])
    in_queue[source] = True
    count = [0] * n  # For negative cycle detection

    while queue:
        u = queue.popleft()
        in_queue[u] = False

        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                if not in_queue[v]:
                    queue.append(v)
                    in_queue[v] = True
                    count[v] += 1
                    if count[v] >= n:
                        return None  # Negative cycle

    return dist
```

---

## 7.20 Bidirectional BFS 🟡

Run BFS from both start and end simultaneously. Meet in the middle.

```python
def bidirectional_bfs(graph, start, end):
    if start == end:
        return [start]

    front_visited = {start: [start]}
    back_visited = {end: [end]}
    front_queue = deque([start])
    back_queue = deque([end])

    while front_queue and back_queue:
        # Expand front
        for _ in range(len(front_queue)):
            node = front_queue.popleft()
            for neighbor in graph[node]:
                if neighbor in back_visited:
                    return front_visited[node] + back_visited[neighbor][::-1]
                if neighbor not in front_visited:
                    front_visited[neighbor] = front_visited[node] + [neighbor]
                    front_queue.append(neighbor)

        # Expand back
        for _ in range(len(back_queue)):
            node = back_queue.popleft()
            for neighbor in graph[node]:
                if neighbor in front_visited:
                    return front_visited[neighbor] + back_visited[node][::-1]
                if neighbor not in back_visited:
                    back_visited[neighbor] = back_visited[node] + [neighbor]
                    back_queue.append(neighbor)

    return None
```

**Advantage:** Explores O(b^(d/2)) nodes instead of O(b^d) where b = branching factor, d = depth.

---

## 7.21 Traveling Salesman Problem (TSP) 🔴

Find the shortest route visiting all cities exactly once and returning to the start. **NP-hard.**

```python
# DP with bitmask — O(2ⁿ × n²)
def tsp(dist, n):
    """dist[i][j] = distance from city i to j"""
    INF = float('inf')
    dp = [[INF] * n for _ in range(1 << n)]
    dp[1][0] = 0  # Start at city 0

    for mask in range(1 << n):
        for u in range(n):
            if dp[mask][u] == INF:
                continue
            for v in range(n):
                if mask & (1 << v):
                    continue
                new_mask = mask | (1 << v)
                dp[new_mask][v] = min(dp[new_mask][v], dp[mask][u] + dist[u][v])

    full_mask = (1 << n) - 1
    return min(dp[full_mask][u] + dist[u][0] for u in range(n))
```

---

## 7.22 Strongly Connected Components Condensation 🟡

After finding SCCs, contract each SCC into a single node. The resulting graph is a DAG.

---

## 7.23 Minimum Cut 🟡

Find the minimum number of edges (or minimum total weight) to disconnect the graph.

For s-t cut: equals max flow (Max-Flow Min-Cut Theorem) — see Chapter 12.

**Stoer-Wagner** algorithm finds global minimum cut in O(VE + V² log V).

---

## 7.24 Transitive Closure 🟡

Find all pairs (u, v) where v is reachable from u. Use Floyd-Warshall with boolean OR instead of addition.

```python
def transitive_closure(graph_matrix, n):
    reach = [row[:] for row in graph_matrix]
    for i in range(n):
        reach[i][i] = True

    for k in range(n):
        for i in range(n):
            for j in range(n):
                reach[i][j] = reach[i][j] or (reach[i][k] and reach[k][j])

    return reach
```

---

## 7.25 Maximum Independent Set / Maximum Clique 🔴

Both NP-hard in general. For special graph classes (trees, bipartite), polynomial algorithms exist.

```python
# Maximum Independent Set on a tree — O(n) with DP
def max_independent_set_tree(adj, root=0):
    n = len(adj)
    dp = [[0, 0] for _ in range(n)]  # [exclude, include]
    visited = [False] * n

    def dfs(u):
        visited[u] = True
        dp[u][1] = 1  # Include u
        for v in adj[u]:
            if not visited[v]:
                dfs(v)
                dp[u][0] += max(dp[v][0], dp[v][1])  # Exclude u: children can be anything
                dp[u][1] += dp[v][0]                   # Include u: children must be excluded

    dfs(root)
    return max(dp[root])
```

---

## LCA (Lowest Common Ancestor) & Binary Lifting 🟡

Find LCA of two nodes in O(log n) per query after O(n log n) preprocessing.

```python
import math

class BinaryLifting:
    """LCA using binary lifting — O(n log n) build, O(log n) per query"""
    def __init__(self, adj, root=0):
        self.n = len(adj)
        self.LOG = max(1, int(math.log2(self.n)) + 1)
        self.depth = [0] * self.n
        self.up = [[0] * self.n for _ in range(self.LOG)]

        # BFS to compute parent and depth
        from collections import deque
        visited = [False] * self.n
        visited[root] = True
        queue = deque([root])
        self.up[0][root] = root

        while queue:
            u = queue.popleft()
            for v in adj[u]:
                if not visited[v]:
                    visited[v] = True
                    self.depth[v] = self.depth[u] + 1
                    self.up[0][v] = u
                    queue.append(v)

        # Fill binary lifting table
        for k in range(1, self.LOG):
            for v in range(self.n):
                self.up[k][v] = self.up[k-1][self.up[k-1][v]]

    def kth_ancestor(self, v, k):
        """Find k-th ancestor of v in O(log n)"""
        for i in range(self.LOG):
            if k & (1 << i):
                v = self.up[i][v]
        return v

    def lca(self, u, v):
        """Find LCA of u and v in O(log n)"""
        if self.depth[u] < self.depth[v]:
            u, v = v, u

        # Lift u to same depth as v
        diff = self.depth[u] - self.depth[v]
        u = self.kth_ancestor(u, diff)

        if u == v:
            return u

        # Binary lift both until they meet
        for k in range(self.LOG - 1, -1, -1):
            if self.up[k][u] != self.up[k][v]:
                u = self.up[k][u]
                v = self.up[k][v]

        return self.up[0][u]

    def distance(self, u, v):
        """Distance between u and v = depth[u] + depth[v] - 2*depth[LCA]"""
        w = self.lca(u, v)
        return self.depth[u] + self.depth[v] - 2 * self.depth[w]

# Usage
adj = [[1,2], [0,3,4], [0,5], [1], [1], [2]]
#       0
#      / \
#     1   2
#    / \   \
#   3   4   5
lca = BinaryLifting(adj, root=0)
print(lca.lca(3, 5))      # 0
print(lca.lca(3, 4))      # 1
print(lca.distance(3, 5))  # 4
```

```
LCA Approaches Summary:
  Method              | Build     | Query  | Space   | Notes
  --------------------|-----------|--------|---------|------------------
  Binary Lifting      | O(n log n)| O(log n)| O(n log n)| Most versatile
  Euler Tour + RMQ    | O(n)      | O(1)   | O(n)    | Fastest queries
  Tarjan's Offline    | O(n α(n)) | O(1)*  | O(n)    | Batch queries only
  Heavy-Light Decomp. | O(n)      | O(log n)| O(n)   | + path queries

* = answers available after processing all queries
```

---

## Stable Matching (Gale-Shapley) 🟡

Solve the Stable Marriage Problem — match n men to n women such that no two people prefer each other over their assigned partners.

```python
def gale_shapley(men_prefs, women_prefs):
    """
    Gale-Shapley algorithm — O(n²)

    men_prefs[i]: ordered list of women, most preferred first
    women_prefs[j]: ordered list of men, most preferred first

    Returns: list where result[man] = woman

    Properties:
    - Always produces stable matching (no blocking pairs)
    - Men-optimal: each man gets his best possible stable partner
    - Women-pessimal: each woman gets her worst stable partner
    """
    n = len(men_prefs)

    # Precompute women's preference rank for O(1) lookup
    women_rank = [[0] * n for _ in range(n)]
    for w in range(n):
        for rank, m in enumerate(women_prefs[w]):
            women_rank[w][m] = rank

    free_men = list(range(n))
    next_proposal = [0] * n  # Next woman to propose to
    women_partner = [-1] * n  # Current partner of each woman
    men_partner = [-1] * n

    while free_men:
        m = free_men.pop()
        w = men_prefs[m][next_proposal[m]]
        next_proposal[m] += 1

        if women_partner[w] == -1:
            # Woman is free — accept
            women_partner[w] = m
            men_partner[m] = w
        elif women_rank[w][m] < women_rank[w][women_partner[w]]:
            # Woman prefers new proposer — switch
            old_m = women_partner[w]
            women_partner[w] = m
            men_partner[m] = w
            men_partner[old_m] = -1
            free_men.append(old_m)
        else:
            # Woman rejects — man stays free
            free_men.append(m)

    return men_partner

# Example
men_prefs = [
    [0, 1, 2],  # Man 0 prefers W0 > W1 > W2
    [1, 0, 2],  # Man 1 prefers W1 > W0 > W2
    [0, 1, 2],  # Man 2 prefers W0 > W1 > W2
]
women_prefs = [
    [1, 0, 2],  # Woman 0 prefers M1 > M0 > M2
    [0, 1, 2],  # Woman 1 prefers M0 > M1 > M2
    [0, 1, 2],  # Woman 2 prefers M0 > M1 > M2
]
print(gale_shapley(men_prefs, women_prefs))  # [0, 1, 2]

# Applications:
# - Medical residency matching (NRMP)
# - College admissions
# - Kidney exchange programs
# - Stable roommates problem (variant)
```

---

## Additional Graph Algorithms

### 2-SAT (Satisfiability) 🟡

Determine if a 2-CNF boolean formula is satisfiable using implication graph + SCC.

```python
class TwoSAT:
    """2-SAT solver using Kosaraju's SCC"""
    def __init__(self, n):
        self.n = n  # number of variables (0-indexed)
        self.adj = [[] for _ in range(2 * n)]
        self.radj = [[] for _ in range(2 * n)]

    def _neg(self, x):
        return x ^ 1

    def _var(self, x, neg=False):
        return 2 * x + (1 if neg else 0)

    def add_clause(self, x, neg_x, y, neg_y):
        """Add clause (x ∨ y). neg_x/neg_y = True means ¬x/¬y"""
        u = self._var(x, neg_x)
        v = self._var(y, neg_y)
        # ¬u → v and ¬v → u
        self.adj[self._neg(u)].append(v)
        self.adj[self._neg(v)].append(u)
        self.radj[v].append(self._neg(u))
        self.radj[u].append(self._neg(v))

    def solve(self):
        n2 = 2 * self.n
        order = []
        visited = [False] * n2
        comp = [-1] * n2

        def dfs1(v):
            stack = [(v, False)]
            while stack:
                node, processed = stack.pop()
                if processed:
                    order.append(node)
                    continue
                if visited[node]:
                    continue
                visited[node] = True
                stack.append((node, True))
                for u in self.adj[node]:
                    if not visited[u]:
                        stack.append((u, False))

        def dfs2(v, c):
            stack = [v]
            while stack:
                node = stack.pop()
                if comp[node] != -1:
                    continue
                comp[node] = c
                for u in self.radj[node]:
                    if comp[u] == -1:
                        stack.append(u)

        for i in range(n2):
            if not visited[i]:
                dfs1(i)

        c = 0
        for v in reversed(order):
            if comp[v] == -1:
                dfs2(v, c)
                c += 1

        # Check satisfiability
        assignment = [False] * self.n
        for i in range(self.n):
            if comp[2*i] == comp[2*i+1]:
                return None  # UNSATISFIABLE
            assignment[i] = comp[2*i] > comp[2*i+1]
        return assignment

# Example: (x0 ∨ x1) ∧ (¬x0 ∨ x2) ∧ (¬x1 ∨ ¬x2)
sat = TwoSAT(3)
sat.add_clause(0, False, 1, False)   # x0 ∨ x1
sat.add_clause(0, True, 2, False)    # ¬x0 ∨ x2
sat.add_clause(1, True, 2, True)     # ¬x1 ∨ ¬x2
print(sat.solve())  # e.g., [True, False, True]
```

### Euler Tour Technique 🟡

Flatten a tree into an array for range queries (LCA, subtree sums, etc.)

```python
def euler_tour(adj, root=0):
    """Returns tin (entry time), tout (exit time), and euler tour order"""
    n = len(adj)
    tin = [0] * n
    tout = [0] * n
    tour = []
    timer = [0]

    def dfs(u, parent):
        tin[u] = timer[0]
        tour.append(u)
        timer[0] += 1
        for v in adj[u]:
            if v != parent:
                dfs(v, u)
                tour.append(u)
                timer[0] += 1
        tout[u] = timer[0] - 1

    dfs(root, -1)
    return tin, tout, tour

# Use cases:
# - Subtree queries: nodes in subtree of u → indices [tin[u], tout[u]]
# - LCA queries: LCA(u,v) = node with minimum depth in tour[tin[u]..tin[v]]
# - Path queries: combine with HLD (Heavy-Light Decomposition)
```

### Yen's K Shortest Paths 🟡

Find K shortest simple paths between two nodes.

```python
import heapq
from copy import deepcopy

def yen_k_shortest(adj, src, dst, K):
    """Find K shortest paths. adj = {node: [(neighbor, weight), ...]}"""
    def dijkstra(adj, src, dst, removed_edges=set(), removed_nodes=set()):
        dist = {src: 0}
        prev = {src: None}
        pq = [(0, src)]
        while pq:
            d, u = heapq.heappop(pq)
            if u == dst:
                path = []
                while u is not None:
                    path.append(u)
                    u = prev[u]
                return path[::-1], d
            if d > dist.get(u, float('inf')):
                continue
            if u in removed_nodes:
                continue
            for v, w in adj.get(u, []):
                if v in removed_nodes or (u, v) in removed_edges:
                    continue
                nd = d + w
                if nd < dist.get(v, float('inf')):
                    dist[v] = nd
                    prev[v] = u
                    heapq.heappush(pq, (nd, v))
        return None, float('inf')

    A = []  # K shortest paths
    B = []  # Candidate paths (min-heap)

    path, cost = dijkstra(adj, src, dst)
    if path is None:
        return []
    A.append((cost, path))

    for k in range(1, K):
        for i in range(len(A[-1][1]) - 1):
            spur_node = A[-1][1][i]
            root_path = A[-1][1][:i+1]

            removed_edges = set()
            for cost_p, p in A:
                if p[:i+1] == root_path:
                    removed_edges.add((p[i], p[i+1]))

            removed_nodes = set(root_path[:-1])
            spur_path, spur_cost = dijkstra(adj, spur_node, dst, removed_edges, removed_nodes)

            if spur_path:
                total_path = root_path[:-1] + spur_path
                # Calculate total cost
                total_cost = 0
                for j in range(len(total_path)-1):
                    for v, w in adj[total_path[j]]:
                        if v == total_path[j+1]:
                            total_cost += w
                            break
                heapq.heappush(B, (total_cost, total_path))

        if not B:
            break
        cost, path = heapq.heappop(B)
        A.append((cost, path))

    return A
```

### PageRank 🟡

Google's original algorithm for ranking web pages.

```python
def pagerank(adj, damping=0.85, iterations=100, tol=1e-6):
    """
    adj: adjacency list {node: [outgoing neighbors]}
    Returns dict of node → PageRank score
    """
    nodes = set(adj.keys())
    for neighbors in adj.values():
        nodes.update(neighbors)
    nodes = list(nodes)
    n = len(nodes)

    rank = {node: 1.0 / n for node in nodes}
    out_degree = {node: len(adj.get(node, [])) for node in nodes}

    for _ in range(iterations):
        new_rank = {}
        for node in nodes:
            incoming_sum = 0
            for other in nodes:
                if node in adj.get(other, []):
                    incoming_sum += rank[other] / max(out_degree[other], 1)
            new_rank[node] = (1 - damping) / n + damping * incoming_sum

        # Check convergence
        diff = sum(abs(new_rank[n] - rank[n]) for n in nodes)
        rank = new_rank
        if diff < tol:
            break

    return rank
```

### Edmonds' Blossom Algorithm (General Matching) 🔴

Find maximum matching in general (non-bipartite) graphs. Handles odd-length cycles (blossoms) by shrinking them.

```
Concept:
1. Start with empty matching
2. Find augmenting paths using BFS/DFS
3. When odd cycle (blossom) found, shrink it to single vertex
4. Continue finding augmenting paths on contracted graph
5. Expand blossoms to recover actual matching

Time: O(V³) or O(V·E) with optimizations
```

### Chinese Postman Problem 🟡

Find minimum cost walk that traverses every edge at least once.

```python
def chinese_postman(n, edges):
    """
    For undirected graph:
    1. Find all odd-degree vertices
    2. Find min-weight perfect matching on odd vertices
    3. Add matched edges → all degrees even → Euler circuit exists
    """
    from itertools import combinations
    import heapq

    adj = [[] for _ in range(n)]
    total_weight = 0
    degree = [0] * n

    for u, v, w in edges:
        adj[u].append((v, w))
        adj[v].append((u, w))
        degree[u] += 1
        degree[v] += 1
        total_weight += w

    # Find odd-degree vertices
    odd = [v for v in range(n) if degree[v] % 2 == 1]

    if not odd:
        return total_weight  # Already Eulerian

    # Find shortest paths between all pairs of odd vertices (Dijkstra)
    def dijkstra(src):
        dist = [float('inf')] * n
        dist[src] = 0
        pq = [(0, src)]
        while pq:
            d, u = heapq.heappop(pq)
            if d > dist[u]:
                continue
            for v, w in adj[u]:
                if d + w < dist[v]:
                    dist[v] = d + w
                    heapq.heappush(pq, (dist[v], v))
        return dist

    dists = {v: dijkstra(v) for v in odd}

    # Min-weight perfect matching on odd vertices (brute force for small sets)
    def min_matching(vertices):
        if len(vertices) == 0:
            return 0
        if len(vertices) == 2:
            return dists[vertices[0]][vertices[1]]
        u = vertices[0]
        rest = vertices[1:]
        best = float('inf')
        for i, v in enumerate(rest):
            remaining = rest[:i] + rest[i+1:]
            cost = dists[u][v] + min_matching(remaining)
            best = min(best, cost)
        return best

    return total_weight + min_matching(odd)
```

### Community Detection (Louvain Method) 🟡

Detect communities in networks by maximizing modularity.

```python
def louvain_communities(adj, weights=None):
    """Simplified Louvain community detection"""
    n = len(adj)
    community = list(range(n))  # Each node starts in its own community

    # Total edge weight
    m = sum(len(adj[u]) for u in range(n)) / 2
    if m == 0:
        return community

    # Degree of each node
    degree = [len(adj[u]) for u in range(n)]

    improved = True
    while improved:
        improved = False
        for u in range(n):
            best_community = community[u]
            best_gain = 0

            # Try moving u to each neighbor's community
            neighbor_communities = set(community[v] for v in adj[u])
            for c in neighbor_communities:
                # Calculate modularity gain
                ki = degree[u]
                ki_in = sum(1 for v in adj[u] if community[v] == c)
                sigma_tot = sum(degree[v] for v in range(n) if community[v] == c)

                gain = ki_in / m - (sigma_tot * ki) / (2 * m * m)
                if gain > best_gain:
                    best_gain = gain
                    best_community = c

            if best_community != community[u]:
                community[u] = best_community
                improved = True

    return community
```

### Planarity Testing (Boyer-Myrvold) 🔴

Test if a graph can be drawn on a plane without edge crossings.

```
Key concepts:
- A graph is planar iff it contains no K₅ or K₃,₃ subdivision (Kuratowski's theorem)
- Euler's formula for planar graphs: V - E + F = 2
- For simple planar graphs: E ≤ 3V - 6

Quick check: If E > 3V - 6, the graph is NOT planar.

Boyer-Myrvold algorithm runs in O(V) time.

Applications:
- Circuit board layout
- Geographic map rendering
- Graph drawing / visualization
```

```python
def quick_planarity_check(V, E):
    """Necessary (not sufficient) condition for planarity"""
    if V < 3:
        return True
    return E <= 3 * V - 6
```

### Tree Isomorphism 🟡

Check if two unrooted trees are isomorphic using canonical form (AHU algorithm).

```python
def tree_isomorphism(adj1, adj2):
    """Check if two trees are isomorphic — O(n log n)"""
    if len(adj1) != len(adj2):
        return False
    n = len(adj1)
    if n <= 1:
        return True

    def find_centers(adj):
        """Center of tree = nodes remaining after iterative leaf removal"""
        n = len(adj)
        degree = [len(adj[i]) for i in range(n)]
        leaves = [i for i in range(n) if degree[i] <= 1]
        remaining = n
        while remaining > 2:
            new_leaves = []
            for leaf in leaves:
                for neighbor in adj[leaf]:
                    degree[neighbor] -= 1
                    if degree[neighbor] == 1:
                        new_leaves.append(neighbor)
            remaining -= len(leaves)
            leaves = new_leaves
        return leaves

    def canonical_form(adj, root):
        """Get canonical string representation of rooted tree"""
        def dfs(u, parent):
            children_forms = []
            for v in adj[u]:
                if v != parent:
                    children_forms.append(dfs(v, u))
            children_forms.sort()
            return "(" + "".join(children_forms) + ")"
        return dfs(root, -1)

    centers1 = find_centers(adj1)
    centers2 = find_centers(adj2)

    forms1 = {canonical_form(adj1, c) for c in centers1}
    forms2 = {canonical_form(adj2, c) for c in centers2}

    return bool(forms1 & forms2)
```

### Spectral Graph Theory Basics 🔴

Using eigenvalues of graph matrices for structural insights.

```python
import numpy as np

def spectral_analysis(adj_matrix):
    """Basic spectral analysis of a graph"""
    A = np.array(adj_matrix, dtype=float)
    n = len(A)

    # Degree matrix
    D = np.diag(A.sum(axis=1))

    # Laplacian L = D - A
    L = D - A

    # Normalized Laplacian
    D_inv_sqrt = np.diag(1.0 / np.sqrt(np.maximum(D.diagonal(), 1)))
    L_norm = D_inv_sqrt @ L @ D_inv_sqrt

    # Eigenvalues and eigenvectors
    eigenvalues, eigenvectors = np.linalg.eigh(L)

    # Number of connected components = multiplicity of eigenvalue 0
    num_components = np.sum(np.abs(eigenvalues) < 1e-10)

    # Fiedler value (algebraic connectivity) = second smallest eigenvalue
    sorted_eigs = np.sort(eigenvalues)
    fiedler_value = sorted_eigs[1] if n > 1 else 0

    # Spectral clustering using Fiedler vector
    fiedler_vector = eigenvectors[:, np.argsort(eigenvalues)[1]]
    partition = [1 if x >= 0 else 0 for x in fiedler_vector]

    return {
        'num_components': num_components,
        'fiedler_value': fiedler_value,
        'spectral_partition': partition,
        'eigenvalues': sorted_eigs
    }
```

---

## Algorithm Selection Guide

```
Shortest path (single source, unweighted):     BFS
Shortest path (single source, non-negative):   Dijkstra
Shortest path (single source, negative ok):    Bellman-Ford / SPFA
Shortest path (all pairs, dense):              Floyd-Warshall
Shortest path (all pairs, sparse):             Johnson's
Minimum Spanning Tree (sparse):                Kruskal's
Minimum Spanning Tree (dense):                 Prim's
Topological ordering:                          Kahn's or DFS-based
Strongly connected components:                 Tarjan's or Kosaraju's
Cycle detection (directed):                    DFS with colors
Cycle detection (undirected):                  Union-Find or DFS
Bipartite check:                               BFS/DFS 2-coloring
Articulation points / bridges:                 Tarjan's variant
Boolean satisfiability (2-CNF):                2-SAT (SCC-based)
General matching:                              Edmonds' Blossom
K shortest paths:                              Yen's algorithm
Web page ranking:                              PageRank
Community detection:                           Louvain method
Planarity testing:                             Boyer-Myrvold
Tree isomorphism:                              AHU canonical form
Euler circuit:                                 Chinese Postman + Hierholzer
Subtree/path queries on trees:                 Euler Tour + HLD
```

---

[← Previous: Sorting & Searching](06-sorting-and-searching.md) | [Next: Dynamic Programming →](08-dynamic-programming.md)
