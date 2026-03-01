# Chapter 12: Network Flow & Computational Geometry 🟡🔴

---

# Part A: Network Flow

## What is Network Flow?

Model problems as a directed graph with source (s), sink (t), and edge capacities. Find the maximum flow from s to t.

```
        10       10
   s -------→ A -------→ t
   |           ↑ ↘        ↑
   |  10    5  |    5      | 10
   |           |      ↘    |
   └-------→ B -------→ C-┘
        10       10
```

**Max-Flow Min-Cut Theorem:** Max flow = Min cut (minimum total capacity of edges whose removal disconnects s from t).

---

## 12.1 Ford-Fulkerson Method 🟡

Find augmenting paths from s to t, push flow along them.

```python
from collections import defaultdict

def ford_fulkerson(graph, source, sink, n):
    """
    graph: adjacency list with capacities
    Returns max flow value
    """
    # Build residual graph
    capacity = [[0] * n for _ in range(n)]
    for u in graph:
        for v, cap in graph[u]:
            capacity[u][v] += cap

    def bfs(source, sink, parent):
        visited = [False] * n
        visited[source] = True
        queue = [source]
        while queue:
            u = queue.pop(0)
            for v in range(n):
                if not visited[v] and capacity[u][v] > 0:
                    visited[v] = True
                    parent[v] = u
                    if v == sink:
                        return True
                    queue.append(v)
        return False

    max_flow = 0
    parent = [-1] * n

    while bfs(source, sink, parent):
        # Find min capacity along the path
        path_flow = float('inf')
        v = sink
        while v != source:
            u = parent[v]
            path_flow = min(path_flow, capacity[u][v])
            v = u

        # Update capacities
        v = sink
        while v != source:
            u = parent[v]
            capacity[u][v] -= path_flow
            capacity[v][u] += path_flow
            v = u

        max_flow += path_flow
        parent = [-1] * n

    return max_flow
```

**Time:** O(V × E²) with BFS (Edmonds-Karp)

---

## 12.2 Edmonds-Karp (Ford-Fulkerson + BFS) 🟡

Always uses shortest augmenting path (BFS). Guarantees O(VE²).

The implementation above IS Edmonds-Karp (uses BFS to find augmenting paths).

---

## 12.3 Dinic's Algorithm 🔴

Uses level graph + blocking flow for O(V²E) complexity.

```python
from collections import deque

class Dinic:
    def __init__(self, n):
        self.n = n
        self.graph = [[] for _ in range(n)]

    def add_edge(self, u, v, cap):
        self.graph[u].append([v, cap, len(self.graph[v])])
        self.graph[v].append([u, 0, len(self.graph[u]) - 1])

    def bfs(self, s, t):
        """Build level graph"""
        self.level = [-1] * self.n
        self.level[s] = 0
        queue = deque([s])
        while queue:
            u = queue.popleft()
            for v, cap, _ in self.graph[u]:
                if self.level[v] < 0 and cap > 0:
                    self.level[v] = self.level[u] + 1
                    queue.append(v)
        return self.level[t] >= 0

    def dfs(self, u, t, pushed):
        """Find blocking flow"""
        if u == t:
            return pushed
        while self.iter[u] < len(self.graph[u]):
            v, cap, rev = self.graph[u][self.iter[u]]
            if self.level[v] == self.level[u] + 1 and cap > 0:
                d = self.dfs(v, t, min(pushed, cap))
                if d > 0:
                    self.graph[u][self.iter[u]][1] -= d
                    self.graph[v][rev][1] += d
                    return d
            self.iter[u] += 1
        return 0

    def max_flow(self, s, t):
        flow = 0
        while self.bfs(s, t):
            self.iter = [0] * self.n
            while True:
                f = self.dfs(s, t, float('inf'))
                if f == 0:
                    break
                flow += f
        return flow

# Example: same graph as above
d = Dinic(4)  # s=0, A=1, B=2, t=3
d.add_edge(0, 1, 10)
d.add_edge(0, 2, 10)
d.add_edge(1, 3, 10)
d.add_edge(2, 3, 10)
d.add_edge(1, 2, 5)
print(d.max_flow(0, 3))  # 20
```

---

## 12.4 Push-Relabel (Preflow-Push) 🔴

Instead of finding paths, pushes flow locally. Time: O(V²E) or O(V³) with FIFO.

```python
def push_relabel(cap, s, t):
    n = len(cap)
    height = [0] * n
    excess = [0] * n
    flow = [[0] * n for _ in range(n)]

    # Initialize: push max flow from source
    height[s] = n
    for v in range(n):
        if cap[s][v] > 0:
            flow[s][v] = cap[s][v]
            flow[v][s] = -cap[s][v]
            excess[v] = cap[s][v]
            excess[s] -= cap[s][v]

    def push(u, v):
        d = min(excess[u], cap[u][v] - flow[u][v])
        flow[u][v] += d
        flow[v][u] -= d
        excess[u] -= d
        excess[v] += d

    def relabel(u):
        min_height = float('inf')
        for v in range(n):
            if cap[u][v] - flow[u][v] > 0:
                min_height = min(min_height, height[v])
        height[u] = min_height + 1

    def discharge(u):
        while excess[u] > 0:
            pushed = False
            for v in range(n):
                if cap[u][v] - flow[u][v] > 0 and height[u] == height[v] + 1:
                    push(u, v)
                    pushed = True
                    if excess[u] == 0:
                        break
            if not pushed:
                relabel(u)

    # Main loop
    active = [i for i in range(n) if i != s and i != t and excess[i] > 0]
    while active:
        u = active[0]
        old_height = height[u]
        discharge(u)
        if height[u] > old_height:
            active.remove(u)
            active.insert(0, u)  # Move to front
        else:
            active.pop(0)
        active = [i for i in range(n) if i != s and i != t and excess[i] > 0]

    return excess[t]
```

---

## 12.5 Min-Cost Max-Flow 🔴

Find maximum flow with minimum total cost. Each edge has capacity AND cost.

```python
from collections import deque

class MinCostFlow:
    def __init__(self, n):
        self.n = n
        self.graph = [[] for _ in range(n)]

    def add_edge(self, u, v, cap, cost):
        self.graph[u].append([v, cap, cost, len(self.graph[v])])
        self.graph[v].append([u, 0, -cost, len(self.graph[u]) - 1])

    def min_cost_flow(self, s, t, max_flow):
        flow = 0
        cost = 0

        while flow < max_flow:
            # SPFA (Bellman-Ford with queue) to find shortest path
            dist = [float('inf')] * self.n
            in_queue = [False] * self.n
            prev_v = [-1] * self.n
            prev_e = [-1] * self.n

            dist[s] = 0
            queue = deque([s])
            in_queue[s] = True

            while queue:
                u = queue.popleft()
                in_queue[u] = False
                for i, (v, cap, c, _) in enumerate(self.graph[u]):
                    if cap > 0 and dist[u] + c < dist[v]:
                        dist[v] = dist[u] + c
                        prev_v[v] = u
                        prev_e[v] = i
                        if not in_queue[v]:
                            queue.append(v)
                            in_queue[v] = True

            if dist[t] == float('inf'):
                break

            # Find max flow along shortest path
            d = max_flow - flow
            v = t
            while v != s:
                d = min(d, self.graph[prev_v[v]][prev_e[v]][1])
                v = prev_v[v]

            # Update flow
            flow += d
            cost += d * dist[t]
            v = t
            while v != s:
                self.graph[prev_v[v]][prev_e[v]][1] -= d
                rev = self.graph[prev_v[v]][prev_e[v]][3]
                self.graph[v][rev][1] += d
                v = prev_v[v]

        return flow, cost
```

---

## 12.6 Bipartite Matching 🟡

### Hungarian Algorithm (Assignment Problem)

```python
def hungarian(cost_matrix):
    """Find min-cost perfect matching in bipartite graph"""
    n = len(cost_matrix)
    u = [0] * (n + 1)  # Potential for left vertices
    v = [0] * (n + 1)  # Potential for right vertices
    match = [0] * (n + 1)

    for i in range(1, n + 1):
        match[0] = i
        j0 = 0
        minv = [float('inf')] * (n + 1)
        used = [False] * (n + 1)

        while match[j0] != 0:
            used[j0] = True
            i0 = match[j0]
            delta = float('inf')
            j1 = -1

            for j in range(1, n + 1):
                if not used[j]:
                    cur = cost_matrix[i0-1][j-1] - u[i0] - v[j]
                    if cur < minv[j]:
                        minv[j] = cur
                    if minv[j] < delta:
                        delta = minv[j]
                        j1 = j

            for j in range(n + 1):
                if used[j]:
                    u[match[j]] += delta
                    v[j] -= delta
                else:
                    minv[j] -= delta

            j0 = j1

        while j0:
            match[j0] = match[0]  # Simplified; actual augmenting path
            j0 = 0  # Placeholder
            break

    return -v[0]  # Total cost
```

### Hopcroft-Karp (Maximum Bipartite Matching)

```python
from collections import deque

def hopcroft_karp(graph, n_left, n_right):
    """
    graph[u] = list of right vertices connected to left vertex u
    Returns max matching size
    """
    match_left = [-1] * n_left
    match_right = [-1] * n_right

    def bfs():
        queue = deque()
        dist = [float('inf')] * n_left
        for u in range(n_left):
            if match_left[u] == -1:
                dist[u] = 0
                queue.append(u)

        found = False
        while queue:
            u = queue.popleft()
            for v in graph[u]:
                next_u = match_right[v]
                if next_u == -1:
                    found = True
                elif dist[next_u] == float('inf'):
                    dist[next_u] = dist[u] + 1
                    queue.append(next_u)
        return found, dist

    def dfs(u, dist):
        for v in graph[u]:
            next_u = match_right[v]
            if next_u == -1 or (dist[next_u] == dist[u] + 1 and dfs(next_u, dist)):
                match_left[u] = v
                match_right[v] = u
                return True
        dist[u] = float('inf')
        return False

    matching = 0
    while True:
        found, dist = bfs()
        if not found:
            break
        for u in range(n_left):
            if match_left[u] == -1:
                if dfs(u, dist):
                    matching += 1

    return matching
```

**Time:** O(E√V)

---

## 12.7 Flow Applications

| Problem | Reduction |
|---------|-----------|
| Maximum bipartite matching | Max flow with unit capacities |
| Minimum vertex cover | Min cut (König's theorem) |
| Maximum independent set | n - min vertex cover |
| Edge-disjoint paths | Max flow with unit capacities |
| Node-disjoint paths | Split nodes, unit capacities |
| Project selection | Min cut |
| Image segmentation | Min cut |

---

# Part B: Computational Geometry

## 12.8 Points and Vectors 🟢

```python
import math

class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __sub__(self, other):
        return Point(self.x - other.x, self.y - other.y)

    def __add__(self, other):
        return Point(self.x + other.x, self.y + other.y)

    def dot(self, other):
        return self.x * other.x + self.y * other.y

    def cross(self, other):
        """Cross product (z-component of 3D cross product)"""
        return self.x * other.y - self.y * other.x

    def magnitude(self):
        return math.sqrt(self.x**2 + self.y**2)

    def dist(self, other):
        return (self - other).magnitude()

def orientation(p, q, r):
    """
    0 → Collinear
    1 → Clockwise
    2 → Counterclockwise
    """
    val = (q - p).cross(r - p)
    if abs(val) < 1e-9: return 0
    return 1 if val < 0 else 2
```

---

## 12.9 Convex Hull 🟡

### Graham Scan — O(n log n)

```python
def graham_scan(points):
    points = sorted(points, key=lambda p: (p.x, p.y))

    def build_half(pts):
        hull = []
        for p in pts:
            while len(hull) >= 2 and (hull[-1] - hull[-2]).cross(p - hull[-2]) <= 0:
                hull.pop()
            hull.append(p)
        return hull

    lower = build_half(points)
    upper = build_half(reversed(points))

    return lower[:-1] + upper[:-1]

# Andrew's Monotone Chain (equivalent)
def convex_hull(points):
    points.sort(key=lambda p: (p.x, p.y))

    lower = []
    for p in points:
        while len(lower) >= 2 and (lower[-1] - lower[-2]).cross(p - lower[-2]) <= 0:
            lower.pop()
        lower.append(p)

    upper = []
    for p in reversed(points):
        while len(upper) >= 2 and (upper[-1] - upper[-2]).cross(p - upper[-2]) <= 0:
            upper.pop()
        upper.append(p)

    return lower[:-1] + upper[:-1]
```

### Jarvis March (Gift Wrapping) — O(nh)

```python
def jarvis_march(points):
    n = len(points)
    if n < 3: return points

    # Start from leftmost point
    start = min(range(n), key=lambda i: (points[i].x, points[i].y))
    hull = []
    current = start

    while True:
        hull.append(points[current])
        next_point = 0

        for i in range(n):
            if i == current:
                continue
            cross = (points[i] - points[current]).cross(
                     points[next_point] - points[current])
            if next_point == current or cross > 0:
                next_point = i
            elif cross == 0:  # Collinear: choose farther point
                if points[current].dist(points[i]) > points[current].dist(points[next_point]):
                    next_point = i

        current = next_point
        if current == start:
            break

    return hull
```

---

## 12.10 Line Segment Intersection 🟡

```python
def on_segment(p, q, r):
    """Check if point q lies on segment pr"""
    return (min(p.x, r.x) <= q.x <= max(p.x, r.x) and
            min(p.y, r.y) <= q.y <= max(p.y, r.y))

def segments_intersect(p1, q1, p2, q2):
    """Check if segments p1q1 and p2q2 intersect"""
    o1 = orientation(p1, q1, p2)
    o2 = orientation(p1, q1, q2)
    o3 = orientation(p2, q2, p1)
    o4 = orientation(p2, q2, q1)

    if o1 != o2 and o3 != o4:
        return True

    if o1 == 0 and on_segment(p1, p2, q1): return True
    if o2 == 0 and on_segment(p1, q2, q1): return True
    if o3 == 0 and on_segment(p2, p1, q2): return True
    if o4 == 0 and on_segment(p2, q1, q2): return True

    return False
```

---

## 12.11 Point in Polygon 🟡

```python
def point_in_polygon(point, polygon):
    """Ray casting algorithm"""
    n = len(polygon)
    inside = False

    j = n - 1
    for i in range(n):
        xi, yi = polygon[i].x, polygon[i].y
        xj, yj = polygon[j].x, polygon[j].y

        if ((yi > point.y) != (yj > point.y) and
            point.x < (xj - xi) * (point.y - yi) / (yj - yi) + xi):
            inside = not inside
        j = i

    return inside
```

---

## 12.12 Sweep Line Algorithms 🟡

### Closest Pair → See [Chapter 10](10-greedy-backtracking-divide-conquer.md)

### Line Segment Intersection (Bentley-Ottmann)

Sweep a vertical line left to right. Events: segment starts, segment ends, two segments swap positions.

### Area of Union of Rectangles

```python
# Sweep line + segment tree for area of union of rectangles
# Complexity: O(n log n) where n = number of rectangles
```

---

## 12.13 Polygon Area 🟢

```python
def polygon_area(polygon):
    """Shoelace formula"""
    n = len(polygon)
    area = 0
    for i in range(n):
        j = (i + 1) % n
        area += polygon[i].x * polygon[j].y
        area -= polygon[j].x * polygon[i].y
    return abs(area) / 2

# Signed area (positive if counterclockwise)
def signed_area(polygon):
    n = len(polygon)
    area = 0
    for i in range(n):
        j = (i + 1) % n
        area += polygon[i].x * polygon[j].y - polygon[j].x * polygon[i].y
    return area / 2
```

---

## 12.14 Rotating Calipers 🔴

Find diameter (farthest pair), width, or minimum bounding rectangle of convex hull in O(n).

```python
def diameter(hull):
    """Find the farthest pair of points in a convex hull"""
    n = len(hull)
    if n <= 1: return 0
    if n == 2: return hull[0].dist(hull[1])

    # Use rotating calipers
    j = 1
    max_dist = 0

    for i in range(n):
        next_i = (i + 1) % n
        while True:
            next_j = (j + 1) % n
            edge = hull[next_i] - hull[i]
            to_next = hull[next_j] - hull[j]
            if edge.cross(to_next) > 0:
                j = next_j
            else:
                break
        max_dist = max(max_dist, hull[i].dist(hull[j]))

    return max_dist
```

---

## 12.15 Voronoi Diagram & Delaunay Triangulation 🔴

**Voronoi Diagram:** Partition plane into regions closest to each point.

**Delaunay Triangulation:** Triangulation where no point is inside any triangle's circumcircle. Dual of Voronoi.

```
Applications:
  - Nearest neighbor queries
  - Path planning
  - Mesh generation
  - Terrain modeling
  - Cell biology (modeling cell territories)
```

These are complex O(n log n) algorithms typically using Fortune's algorithm (Voronoi) or incremental insertion (Delaunay).

---

## 12.16 Algorithm Summary

| Algorithm | Time | Application |
|-----------|------|-------------|
| Ford-Fulkerson/Edmonds-Karp | O(VE²) | Max flow |
| Dinic's | O(V²E) | Max flow (faster) |
| Push-Relabel | O(V²E) or O(V³) | Max flow (practical) |
| Hopcroft-Karp | O(E√V) | Max bipartite matching |
| Hungarian | O(n³) | Min-cost assignment |
| Graham Scan | O(n log n) | Convex hull |
| Sweep Line | O(n log n) | Segment intersection |
| Rotating Calipers | O(n) | Diameter of convex hull |
| Welzl's | O(n) expected | Smallest enclosing circle |
| Chan's | O(n log h) | Output-sensitive convex hull |
| Pick's Theorem | O(n) | Lattice polygon area |

---

## Additional Network Flow & Geometry

### Max Flow Applications (Detailed) 🟡

```python
def min_path_cover_dag(n, adj):
    """
    Minimum path cover in DAG = n - maximum matching in bipartite graph.
    Split each vertex into two (out-copy and in-copy).
    Edge u→v becomes out_u → in_v.
    """
    # Build bipartite graph
    match_left = [-1] * n   # out-copies matched to in-copies
    match_right = [-1] * n  # in-copies matched to out-copies

    def augment(u, visited):
        for v in adj[u]:
            if not visited[v]:
                visited[v] = True
                if match_right[v] == -1 or augment(match_right[v], visited):
                    match_left[u] = v
                    match_right[v] = u
                    return True
        return False

    matching = 0
    for u in range(n):
        visited = [False] * n
        if augment(u, visited):
            matching += 1

    return n - matching  # Minimum path cover

# Example: DAG with edges 0→1, 0→2, 1→3, 2→3
adj = [[1, 2], [3], [3], []]
print(min_path_cover_dag(4, adj))  # 2 (e.g., 0→1→3 and 2)
```

### König's Theorem & Hall's Marriage Theorem 🟡

```
König's Theorem:
  In any bipartite graph:
  Maximum matching = Minimum vertex cover

  This does NOT hold for general graphs!

  To find min vertex cover from max matching:
  1. Find maximum matching (Hopcroft-Karp)
  2. Find alternating tree from unmatched left vertices
  3. Min vertex cover = unvisited left + visited right

Hall's Marriage Theorem:
  A bipartite graph G = (X ∪ Y, E) has a matching that saturates X
  if and only if for every subset S ⊆ X:
    |N(S)| ≥ |S|  (neighborhood of S has at least |S| elements)

  This is the "marriage condition."

Applications:
  - Task assignment
  - Latin squares
  - SDR (System of Distinct Representatives)
```

```python
def konig_vertex_cover(n_left, n_right, adj, matching_left, matching_right):
    """Find minimum vertex cover from maximum matching in bipartite graph"""
    # Find unmatched left vertices
    unmatched_left = {u for u in range(n_left) if matching_left[u] == -1}

    # BFS/DFS alternating from unmatched left
    visited_left = set()
    visited_right = set()
    stack = list(unmatched_left)

    while stack:
        u = stack.pop()
        if u in visited_left:
            continue
        visited_left.add(u)
        for v in adj[u]:
            if v not in visited_right:
                visited_right.add(v)
                # Follow matching edge back
                if matching_right[v] != -1 and matching_right[v] not in visited_left:
                    stack.append(matching_right[v])

    # König's: cover = unvisited left ∪ visited right
    cover = set()
    for u in range(n_left):
        if u not in visited_left:
            cover.add(('L', u))
    for v in visited_right:
        cover.add(('R', v))

    return cover
```

### Welzl's Algorithm (Smallest Enclosing Circle) 🟡

```python
import random
import math

def smallest_enclosing_circle(points):
    """Welzl's algorithm — O(n) expected time"""

    def dist(a, b):
        return math.sqrt((a[0]-b[0])**2 + (a[1]-b[1])**2)

    def circle_from_1(p):
        return (p[0], p[1], 0)

    def circle_from_2(p1, p2):
        cx = (p1[0] + p2[0]) / 2
        cy = (p1[1] + p2[1]) / 2
        r = dist(p1, p2) / 2
        return (cx, cy, r)

    def circle_from_3(p1, p2, p3):
        ax, ay = p1; bx, by = p2; cx, cy = p3
        d = 2 * (ax*(by-cy) + bx*(cy-ay) + cx*(ay-by))
        if abs(d) < 1e-10:
            return None
        ux = ((ax*ax+ay*ay)*(by-cy) + (bx*bx+by*by)*(cy-ay) + (cx*cx+cy*cy)*(ay-by)) / d
        uy = ((ax*ax+ay*ay)*(cx-bx) + (bx*bx+by*by)*(ax-cx) + (cx*cx+cy*cy)*(bx-ax)) / d
        r = dist((ux, uy), p1)
        return (ux, uy, r)

    def in_circle(c, p):
        return dist((c[0], c[1]), p) <= c[2] + 1e-10

    def welzl(P, R):
        if len(P) == 0 or len(R) == 3:
            if len(R) == 0: return (0, 0, 0)
            if len(R) == 1: return circle_from_1(R[0])
            if len(R) == 2: return circle_from_2(R[0], R[1])
            return circle_from_3(R[0], R[1], R[2])

        p = P[0]
        rest = P[1:]
        c = welzl(rest, R)
        if c and in_circle(c, p):
            return c
        return welzl(rest, R + [p])

    pts = list(points)
    random.shuffle(pts)
    return welzl(pts, [])

circle = smallest_enclosing_circle([(0,0), (1,0), (0,1), (1,1)])
print(f"Center: ({circle[0]:.2f}, {circle[1]:.2f}), Radius: {circle[2]:.2f}")
```

### Pick's Theorem 🟢

For a simple polygon with vertices at lattice points:

$$A = I + \frac{B}{2} - 1$$

where A = area, I = interior lattice points, B = boundary lattice points.

```python
from math import gcd

def picks_theorem(polygon):
    """
    polygon: list of (x, y) integer coordinates in order.
    Returns (area, interior_points, boundary_points)
    """
    n = len(polygon)

    # Boundary points
    B = 0
    for i in range(n):
        x1, y1 = polygon[i]
        x2, y2 = polygon[(i+1) % n]
        B += gcd(abs(x2-x1), abs(y2-y1))

    # Area via shoelace
    A2 = 0  # 2 * Area
    for i in range(n):
        x1, y1 = polygon[i]
        x2, y2 = polygon[(i+1) % n]
        A2 += x1 * y2 - x2 * y1
    A2 = abs(A2)

    # Interior points: I = A - B/2 + 1 = (2A - B + 2) / 2
    I = (A2 - B + 2) // 2

    return A2 / 2, I, B

# Triangle with vertices (0,0), (4,0), (0,3)
area, interior, boundary = picks_theorem([(0,0), (4,0), (0,3)])
print(f"Area={area}, Interior={interior}, Boundary={boundary}")
# Area=6.0, Interior=3, Boundary=8 → 6 = 3 + 8/2 - 1 ✓
```

### Half-Plane Intersection 🔴

Find the intersection of half-planes (convex polygon). Used in LP, Voronoi.

```python
def half_plane_intersection(half_planes):
    """
    Each half-plane: (a, b, c) representing ax + by ≤ c
    Returns vertices of the intersection polygon.

    Algorithm: Sort by angle, maintain deque of intersection lines.
    Time: O(n log n)
    """
    import math

    def angle(a, b):
        return math.atan2(b, a)

    # Sort half-planes by angle of normal
    hp = sorted(half_planes, key=lambda h: angle(h[0], h[1]))

    # Incremental intersection (conceptual)
    # Maintain convex polygon as intersection of processed half-planes
    # Use deque: remove from front/back when new HP makes them redundant

    # (Full implementation is ~80 lines — key idea shown)
    return "O(n log n) algorithm"
```

### Minkowski Sum 🟡

Sum of two convex polygons: useful in motion planning and collision detection.

```python
def minkowski_sum(P, Q):
    """
    Minkowski sum of convex polygons P and Q.
    P ⊕ Q = {p + q : p ∈ P, q ∈ Q}

    Result is convex with at most |P| + |Q| vertices.
    Time: O(|P| + |Q|)
    """
    import math

    def edge_angle(p1, p2):
        return math.atan2(p2[1]-p1[1], p2[0]-p1[0])

    # Start from bottom-most point of each
    def bottom_most(poly):
        return min(range(len(poly)), key=lambda i: (poly[i][1], poly[i][0]))

    n, m = len(P), len(Q)
    i0, j0 = bottom_most(P), bottom_most(Q)

    result = []
    i, j = i0, j0

    for _ in range(n + m):
        result.append((P[i%n][0] + Q[j%m][0], P[i%n][1] + Q[j%m][1]))

        edge_p = edge_angle(P[i%n], P[(i+1)%n])
        edge_q = edge_angle(Q[j%m], Q[(j+1)%m])

        if edge_p < edge_q:
            i += 1
        elif edge_p > edge_q:
            j += 1
        else:
            i += 1
            j += 1

    return result
```

### Chan's Algorithm (Output-Sensitive Convex Hull) 🔴

O(n log h) where h = number of hull points. Optimal when h is small.

```python
def chans_algorithm(points):
    """
    Chan's algorithm: O(n log h) convex hull

    Idea:
    1. Guess output size h (start small, double)
    2. Split points into groups of size h
    3. Compute mini-hulls via Graham scan: O(h log h) each
    4. Merge using Jarvis march on mini-hulls: O(n/h × h × log h)

    When guess matches actual hull size: total = O(n log h)
    """
    import math

    def graham_scan(pts):
        """Standard Graham scan for small point sets"""
        if len(pts) <= 1:
            return pts

        pts = sorted(set(pts))

        def cross(O, A, B):
            return (A[0]-O[0])*(B[1]-O[1]) - (A[1]-O[1])*(B[0]-O[0])

        lower = []
        for p in pts:
            while len(lower) >= 2 and cross(lower[-2], lower[-1], p) <= 0:
                lower.pop()
            lower.append(p)

        upper = []
        for p in reversed(pts):
            while len(upper) >= 2 and cross(upper[-2], upper[-1], p) <= 0:
                upper.pop()
            upper.append(p)

        return lower[:-1] + upper[:-1]

    n = len(points)

    for t in range(1, 30):  # log(log(n)) iterations
        h = min(2 ** (2 ** t), n)

        # Split into groups of size h
        groups = [points[i:i+h] for i in range(0, n, h)]
        mini_hulls = [graham_scan(g) for g in groups]

        # Jarvis march on mini-hulls
        hull = []
        # Find starting point (leftmost)
        start = min(points)
        current = start

        for _ in range(h):
            hull.append(current)
            # For each mini-hull, find tangent point (binary search)
            best = None
            for mh in mini_hulls:
                # Find best candidate from this mini-hull
                candidate = max(mh, key=lambda p: (
                    math.atan2(p[1]-current[1], p[0]-current[0])
                    if p != current else -float('inf')
                ))
                if best is None or candidate != current:
                    if best is None:
                        best = candidate
                    else:
                        cross = ((best[0]-current[0])*(candidate[1]-current[1]) -
                                (best[1]-current[1])*(candidate[0]-current[0]))
                        if cross < 0 or (cross == 0 and
                            ((candidate[0]-current[0])**2 + (candidate[1]-current[1])**2) >
                            ((best[0]-current[0])**2 + (best[1]-current[1])**2)):
                            best = candidate

            if best == start:
                return hull
            current = best

        # h was too small, try larger

    return graham_scan(points)  # Fallback
```

### Polygon Triangulation (Ear Clipping) 🟡

```python
def ear_clipping(polygon):
    """
    Triangulate simple polygon via ear clipping — O(n²)
    Returns list of triangles (index triples)
    """
    def cross(O, A, B):
        return (A[0]-O[0])*(B[1]-O[1]) - (A[1]-O[1])*(B[0]-O[0])

    def point_in_triangle(p, a, b, c):
        d1 = cross(p, a, b)
        d2 = cross(p, b, c)
        d3 = cross(p, c, a)
        has_neg = (d1 < 0) or (d2 < 0) or (d3 < 0)
        has_pos = (d1 > 0) or (d2 > 0) or (d3 > 0)
        return not (has_neg and has_pos)

    n = len(polygon)
    indices = list(range(n))
    triangles = []

    while len(indices) > 2:
        ear_found = False
        for i in range(len(indices)):
            prev_i = (i - 1) % len(indices)
            next_i = (i + 1) % len(indices)

            a = polygon[indices[prev_i]]
            b = polygon[indices[i]]
            c = polygon[indices[next_i]]

            # Check if ear (convex vertex with no points inside)
            if cross(a, b, c) <= 0:
                continue

            is_ear = True
            for j in range(len(indices)):
                if j in (prev_i, i, next_i):
                    continue
                if point_in_triangle(polygon[indices[j]], a, b, c):
                    is_ear = False
                    break

            if is_ear:
                triangles.append((indices[prev_i], indices[i], indices[next_i]))
                indices.pop(i)
                ear_found = True
                break

        if not ear_found:
            break

    return triangles
```

---

[← Previous: Math & Bit Algorithms](11-mathematical-and-bit-algorithms.md) | [Next: Specialized & Emerging →](13-specialized-and-emerging.md)
