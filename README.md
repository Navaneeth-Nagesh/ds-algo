# The Complete Guide to Data Structures & Algorithms

## Your Ultimate Programming Reference

Welcome to the most comprehensive guide to every data structure and algorithm in the programming universe. This resource is organized from fundamentals to advanced topics, with clear explanations, visual diagrams, complexity analysis, and code examples in Python.

---

## How to Use This Guide

1. **Start with fundamentals** if you're new — complexity analysis is the language of algorithms
2. **Follow the numbered order** — each topic builds on previous ones
3. **Read the examples** — every concept includes working code
4. **Understand the "When to Use"** sections — knowing *when* is as important as knowing *how*

---

## Table of Contents

### Part 1: Foundations
| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 01 | Complexity Analysis & Fundamentals | [01-complexity-and-fundamentals.md](01-complexity-and-fundamentals.md) | Big O/Θ/Ω, Amortized Analysis, Complexity Classes (P, NP, co-NP, BPP, PSPACE, #P), FPT, Rice's Theorem, Akra-Bazzi |

### Part 2: Data Structures
| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 02 | Linear Data Structures | [02-data-structures-linear.md](02-data-structures-linear.md) | Arrays, Strings, Linked Lists, Stacks, Queues, Deques, Sliding Window/Two-Pointer Templates, Trapping Rain Water, Matrix Patterns |
| 03 | Trees | [03-data-structures-trees.md](03-data-structures-trees.md) | 35+ tree types: BST, AVL, Red-Black, B-Trees, Tries, Segment Trees, Fenwick, Splay, Treap, K-D, Quadtree, R-Tree, AA, Ball Tree, Range Tree |
| 04 | Hash Tables, Heaps & Graphs | [04-data-structures-hash-heap-graph.md](04-data-structures-hash-heap-graph.md) | Hash Maps, Universal/Perfect Hashing, Cuckoo, Binary/Fibonacci/Soft Heaps, Merge K Sorted, Median Stream, Graph Representations |
| 05 | Advanced & Probabilistic Structures | [05-data-structures-advanced.md](05-data-structures-advanced.md) | Skip Lists, Bloom/Xor Filters, Union-Find, LRU Cache, Persistent Structures, LSM-Tree, ART, Retroactive DS |

### Part 3: Algorithms
| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 06 | Sorting & Searching | [06-sorting-and-searching.md](06-sorting-and-searching.md) | 30+ sorts, 15+ searches, Dutch National Flag, Fisher-Yates Shuffle, Binary Search Patterns, Reservoir Sampling |
| 07 | Graph Algorithms | [07-graph-algorithms.md](07-graph-algorithms.md) | BFS, DFS, Dijkstra, MST, SCC, 2-SAT, PageRank, Edmonds' Blossom, Yen's K-Paths, Euler Tour, Spectral Graph Theory |
| 08 | Dynamic Programming | [08-dynamic-programming.md](08-dynamic-programming.md) | Classic DP, Bitmask/Digit/Tree DP, House Robber, Stock Buy/Sell, Word Break, CHT/Li Chao, Hirschberg, Steiner Tree |
| 09 | String Algorithms | [09-string-algorithms.md](09-string-algorithms.md) | KMP, Boyer-Moore, Aho-Corasick, Suffix Arrays, Eertree, Lyndon, Bitap, FM-Index, Sequence Alignment |
| 10 | Greedy, Backtracking & Divide and Conquer | [10-greedy-backtracking-divide-conquer.md](10-greedy-backtracking-divide-conquer.md) | Activity Selection, Huffman, Matroid Theory, Weighted Interval, Branch & Bound, Algorithm X/DLX, Knight's Tour |
| 11 | Mathematical & Bit Manipulation | [11-mathematical-and-bit-algorithms.md](11-mathematical-and-bit-algorithms.md) | Number Theory, FFT, Möbius, Burnside's Lemma, Lagrange Interpolation, Simplex, Stirling Numbers, Generating Functions |
| 12 | Network Flow & Computational Geometry | [12-network-flow-and-geometry.md](12-network-flow-and-geometry.md) | Max Flow, Min Cut, König's Theorem, Convex Hull (Graham/Chan), Welzl's, Pick's, Minkowski Sum, Polygon Triangulation |
| 13 | Specialized & Emerging Algorithms | [13-specialized-and-emerging.md](13-specialized-and-emerging.md) | Compression, Crypto, Parallel, Quantum, Online Algorithms, Approximation, Game Theory (Minimax/MCTS), Metaheuristics (SA/GA/ACO/PSO) |

### Part 4: System Design & Software Engineering
| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 14 | System Design Fundamentals | [14-system-design-fundamentals.md](14-system-design-fundamentals.md) | Estimation, Scalability, CAP/PACELC, Consistency Models, REST/gRPC/GraphQL, Consistent Hashing, Partitioning, Replication, ACID/BASE, Security, Monitoring |
| 15 | System Design Building Blocks | [15-system-design-building-blocks.md](15-system-design-building-blocks.md) | Load Balancing (L4/L7), Caching (LRU/LFU/Redis), CDN, Message Queues (Kafka), Rate Limiting, Circuit Breakers, Search (Elasticsearch), Microservices, Service Mesh |
| 16 | Distributed Systems | [16-distributed-systems.md](16-distributed-systems.md) | Paxos, Raft, Vector Clocks, Gossip Protocol, MapReduce, Distributed Locking (Redlock), Sagas, CRDTs, CDC, Leader Election, Geo-Distribution |
| 17 | Classic System Designs | [17-system-design-classic.md](17-system-design-classic.md) | 20 designs: URL Shortener, Chat, Feed, Video, Uber, Google Maps, E-Commerce, Google Docs, Social Network, Stock Exchange |
| 18 | Design Patterns (GoF) | [18-design-patterns.md](18-design-patterns.md) | All 23 GoF + Repository, DI Container, Null Object, Object Pool: Factory, Builder, Singleton, Adapter, Observer, Strategy, Command, State |
| 19 | Object-Oriented Design (LLD) | [19-object-oriented-design.md](19-object-oriented-design.md) | SOLID, UML, Parking Lot, Elevator, Chess, Vending Machine, Hotel, Food Delivery, ATM, Movie Ticket, File System, Splitwise |

---

## Quick Reference: Complexity Cheat Sheet

| Data Structure | Access | Search | Insert | Delete | Space |
|----------------|--------|--------|--------|--------|-------|
| Array | O(1) | O(n) | O(n) | O(n) | O(n) |
| Linked List | O(n) | O(n) | O(1) | O(1) | O(n) |
| Stack | O(n) | O(n) | O(1) | O(1) | O(n) |
| Queue | O(n) | O(n) | O(1) | O(1) | O(n) |
| Hash Table | N/A | O(1)* | O(1)* | O(1)* | O(n) |
| BST | O(log n)* | O(log n)* | O(log n)* | O(log n)* | O(n) |
| AVL Tree | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Red-Black Tree | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| B-Tree | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Heap | N/A | O(n) | O(log n) | O(log n) | O(n) |
| Trie | O(m) | O(m) | O(m) | O(m) | O(n·m) |

*\* = average case; worst case may differ*
*m = length of key/string*

| Sorting Algorithm | Best | Average | Worst | Space | Stable? |
|-------------------|------|---------|-------|-------|---------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes |
| Tim Sort | O(n) | O(n log n) | O(n log n) | O(n) | Yes |

---

## Legend

- 🟢 **Beginner** — Start here
- 🟡 **Intermediate** — Build on fundamentals
- 🔴 **Advanced** — Deep expertise required
- 🔮 **Emerging** — Cutting-edge / research topics

---

## Quick Reference: Coding Patterns

```
Pattern                │ Template                              │ When to Use
───────────────────────│───────────────────────────────────────│──────────────────────────────
Fixed Sliding Window   │ maintain window [i-k..i], slide right │ Max sum subarray of size k
Variable Sliding Window│ expand right, shrink left on condition│ Longest substring w/o repeats
Two Pointers (opposite)│ lo=0, hi=n-1, move inward            │ Two Sum (sorted), container
Two Pointers (same dir)│ slow/fast, skip/collect               │ Remove duplicates, linked list
Fast-Slow Pointers     │ slow +1, fast +2                      │ Cycle detection, find middle
Binary Search (sorted) │ lo=0, hi=n-1, bisect                 │ Search rotated, first/last pos
Binary Search (answer) │ lo=min, hi=max, check(mid)           │ Min speed, split array, capacity
BFS (Level-order)      │ queue + level tracking                │ Shortest path, level traversal
DFS (Backtracking)     │ choose → explore → unchoose           │ Permutations, subsets, N-Queens
Monotonic Stack        │ pop while top violates ordering       │ Next greater, largest rectangle
Monotonic Queue        │ deque maintaining order               │ Sliding window max/min
Merge Intervals        │ sort by start, merge overlapping      │ Meeting rooms, interval scheduling
Topological Sort       │ Kahn's (indegree) or DFS post-order  │ Course schedule, build order
Union-Find             │ find(x) with path compression        │ Connected components, Kruskal's
Prefix Sum             │ prefix[i] = sum(arr[0..i])           │ Range sum queries, subarray sum
Dutch National Flag    │ lo/mid/hi three-way partition         │ Sort colors, 3-value arrays
Kadane's Algorithm     │ max_ending_here = max(a[i], prev+a[i])│ Maximum subarray sum
Trie                   │ nested dict/nodes, char-by-char insert│ Auto-complete, word search II
Heap (Top-K)           │ min-heap of size k                    │ Kth largest, top-K frequent
Two Heaps              │ max-heap (lo) + min-heap (hi)         │ Find median from stream
```

---

*Total coverage: 150+ data structures, 250+ algorithms, 20 classic system designs, 27 design patterns, and 14 LLD problems across 19 chapters.*

*Happy learning!*
