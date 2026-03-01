# Chapter 3: Tree Data Structures 🟢🟡🔴

## Overview

Trees are **hierarchical** data structures with a root node and child nodes forming a recursive structure. They're everywhere — file systems, databases, compilers, AI, and more.

```
         Root
        /    \
    Child    Child
    / \        \
  Leaf Leaf   Leaf
```

### Tree Terminology

| Term | Definition |
|------|-----------|
| **Root** | The topmost node (no parent) |
| **Node** | An element in the tree |
| **Edge** | Connection between two nodes |
| **Parent** | Node directly above |
| **Child** | Node directly below |
| **Leaf** | Node with no children |
| **Internal Node** | Node with at least one child |
| **Depth** | Distance from root to node (root = 0) |
| **Height** | Distance from node to deepest leaf |
| **Level** | Same as depth |
| **Subtree** | A node and all its descendants |
| **Degree** | Number of children a node has |
| **Siblings** | Nodes with the same parent |
| **Ancestor** | Any node on the path from root to this node |
| **Descendant** | Any node in this node's subtree |

---

## 3.1 Binary Tree 🟢

Each node has **at most 2 children** (left and right).

```
        1
       / \
      2   3
     / \   \
    4   5   6
```

### Types of Binary Trees

```
Full Binary Tree:          Complete Binary Tree:     Perfect Binary Tree:
Every node has 0 or 2      All levels full except    All levels completely
children                    possibly the last (left   filled
                           aligned)
     1                          1                        1
    / \                        / \                      / \
   2   3                      2   3                    2   3
  / \                        / \ /                    / \ / \
 4   5                      4  5 6                   4  5 6  7

Degenerate (Skewed):       Balanced Binary Tree:
Every node has only         Height difference between
1 child (like a list)       left and right subtrees
                            is at most 1
  1                              4
   \                            / \
    2                          2   6
     \                        / \ / \
      3                      1  3 5  7
       \
        4
```

### Implementation & Traversals

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

# ===== DEPTH-FIRST TRAVERSALS =====

# 1. Inorder (Left → Root → Right) — gives sorted order for BST
def inorder(root):
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

# 2. Preorder (Root → Left → Right) — used to copy/serialize tree
def preorder(root):
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

# 3. Postorder (Left → Right → Root) — used to delete tree
def postorder(root):
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]

# ===== BREADTH-FIRST TRAVERSAL =====

# Level Order (BFS) — visit level by level
from collections import deque

def level_order(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result

# ===== ITERATIVE TRAVERSALS (using explicit stack) =====

def inorder_iterative(root):
    result, stack = [], []
    current = root
    while current or stack:
        while current:
            stack.append(current)
            current = current.left
        current = stack.pop()
        result.append(current.val)
        current = current.right
    return result

# ===== Morris Traversal — O(1) space inorder =====
def morris_inorder(root):
    result = []
    current = root
    while current:
        if not current.left:
            result.append(current.val)
            current = current.right
        else:
            # Find inorder predecessor
            predecessor = current.left
            while predecessor.right and predecessor.right != current:
                predecessor = predecessor.right

            if not predecessor.right:
                # Make current the right child of its predecessor
                predecessor.right = current
                current = current.left
            else:
                # Restore tree structure
                predecessor.right = None
                result.append(current.val)
                current = current.right
    return result

# Example
root = TreeNode(1)
root.left = TreeNode(2)
root.right = TreeNode(3)
root.left.left = TreeNode(4)
root.left.right = TreeNode(5)

print("Inorder:   ", inorder(root))      # [4, 2, 5, 1, 3]
print("Preorder:  ", preorder(root))     # [1, 2, 4, 5, 3]
print("Postorder: ", postorder(root))    # [4, 5, 2, 3, 1]
print("Level:     ", level_order(root))  # [[1], [2, 3], [4, 5]]
```

### Common Binary Tree Operations

```python
def height(root):
    if not root:
        return -1  # or 0, depending on convention
    return 1 + max(height(root.left), height(root.right))

def count_nodes(root):
    if not root:
        return 0
    return 1 + count_nodes(root.left) + count_nodes(root.right)

def is_balanced(root):
    def check(node):
        if not node:
            return 0
        left = check(node.left)
        right = check(node.right)
        if left == -1 or right == -1 or abs(left - right) > 1:
            return -1
        return 1 + max(left, right)
    return check(root) != -1

def diameter(root):
    """Longest path between any two nodes"""
    max_d = [0]
    def depth(node):
        if not node:
            return 0
        l, r = depth(node.left), depth(node.right)
        max_d[0] = max(max_d[0], l + r)
        return 1 + max(l, r)
    depth(root)
    return max_d[0]

def lowest_common_ancestor(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)
    if left and right:
        return root
    return left or right
```

---

## 3.2 Binary Search Tree (BST) 🟢

A binary tree where: **left child < parent < right child** (for all nodes).

```
        8
       / \
      3   10
     / \    \
    1   6   14
       / \  /
      4  7 13

  Inorder traversal gives sorted order: 1, 3, 4, 6, 7, 8, 10, 13, 14
```

### Operations

| Operation | Average | Worst (skewed) |
|-----------|---------|----------------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |
| Min/Max | O(log n) | O(n) |

```python
class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        self.root = self._insert(self.root, val)

    def _insert(self, node, val):
        if not node:
            return TreeNode(val)
        if val < node.val:
            node.left = self._insert(node.left, val)
        elif val > node.val:
            node.right = self._insert(node.right, val)
        return node

    def search(self, val):
        return self._search(self.root, val)

    def _search(self, node, val):
        if not node:
            return False
        if val == node.val:
            return True
        elif val < node.val:
            return self._search(node.left, val)
        else:
            return self._search(node.right, val)

    def delete(self, val):
        self.root = self._delete(self.root, val)

    def _delete(self, node, val):
        if not node:
            return None
        if val < node.val:
            node.left = self._delete(node.left, val)
        elif val > node.val:
            node.right = self._delete(node.right, val)
        else:
            # Node to delete found
            if not node.left:         # Case 1: No left child
                return node.right
            elif not node.right:      # Case 2: No right child
                return node.left
            else:                     # Case 3: Two children
                # Find inorder successor (smallest in right subtree)
                successor = node.right
                while successor.left:
                    successor = successor.left
                node.val = successor.val
                node.right = self._delete(node.right, successor.val)
        return node

    def find_min(self):
        current = self.root
        while current.left:
            current = current.left
        return current.val

    def find_max(self):
        current = self.root
        while current.right:
            current = current.right
        return current.val
```

---

## 3.3 AVL Tree (Self-Balancing BST) 🟡

An AVL tree is a BST where the **height difference** between left and right subtrees of any node is at most 1.

```
Balance Factor = height(left subtree) - height(right subtree)
Valid balance factors: -1, 0, +1
```

### Rotations

```
Right Rotation (when left-heavy):        Left Rotation (when right-heavy):
     z                  y                      z                y
    / \               /   \                   / \             /   \
   y   T4           x      z                T1   y          z     x
  / \       →      / \    / \          →        / \        / \   / \
 x   T3           T1  T2 T3 T4               T2   x      T1 T2 T3 T4
/ \                                               / \
T1 T2                                            T3  T4

Left-Right (double):                    Right-Left (double):
    z         z          x              z        z           x
   / \       / \       /   \           / \      / \        /   \
  y   T4    x   T4    y     z        T1   y   T1  x      z     y
 / \   →   / \    →  / \   / \          / \      / \  → / \   / \
T1  x     y  T3     T1 T2 T3 T4       x  T4    T2  y  T1 T2 T3 T4
   / \   / \                          / \          / \
  T2 T3 T1 T2                       T2  T3       T3  T4
```

```python
class AVLNode:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None
        self.height = 1

class AVLTree:
    def get_height(self, node):
        return node.height if node else 0

    def get_balance(self, node):
        return self.get_height(node.left) - self.get_height(node.right) if node else 0

    def right_rotate(self, z):
        y = z.left
        T3 = y.right

        y.right = z
        z.left = T3

        z.height = 1 + max(self.get_height(z.left), self.get_height(z.right))
        y.height = 1 + max(self.get_height(y.left), self.get_height(y.right))

        return y  # New root

    def left_rotate(self, z):
        y = z.right
        T2 = y.left

        y.left = z
        z.right = T2

        z.height = 1 + max(self.get_height(z.left), self.get_height(z.right))
        y.height = 1 + max(self.get_height(y.left), self.get_height(y.right))

        return y  # New root

    def insert(self, root, val):
        # 1. Normal BST insert
        if not root:
            return AVLNode(val)
        if val < root.val:
            root.left = self.insert(root.left, val)
        elif val > root.val:
            root.right = self.insert(root.right, val)
        else:
            return root  # Duplicates not allowed

        # 2. Update height
        root.height = 1 + max(self.get_height(root.left), self.get_height(root.right))

        # 3. Get balance factor
        balance = self.get_balance(root)

        # 4. Rebalance if needed
        # Left-Left Case
        if balance > 1 and val < root.left.val:
            return self.right_rotate(root)

        # Right-Right Case
        if balance < -1 and val > root.right.val:
            return self.left_rotate(root)

        # Left-Right Case
        if balance > 1 and val > root.left.val:
            root.left = self.left_rotate(root.left)
            return self.right_rotate(root)

        # Right-Left Case
        if balance < -1 and val < root.right.val:
            root.right = self.right_rotate(root.right)
            return self.left_rotate(root)

        return root
```

**All operations: O(log n) guaranteed** — no degenerate cases like plain BST.

---

## 3.4 Red-Black Tree 🟡

A self-balancing BST with **color properties**:

1. Every node is either **red** or **black**
2. The root is **black**
3. Every leaf (NIL) is **black**
4. If a node is **red**, both children are **black** (no two consecutive reds)
5. Every path from a node to its descendant NIL nodes has the same number of black nodes

```
        8(B)
       /    \
     4(R)   12(R)
    / \     / \
  2(B) 6(B) 10(B) 14(B)
```

### Why Red-Black Trees?

- **Guaranteed O(log n)** for search, insert, delete
- **Fewer rotations** than AVL during insertion/deletion (at most 2 for insert, 3 for delete)
- Used in: Java TreeMap, C++ std::map, Linux kernel, etc.

### Red-Black vs AVL

| Feature | AVL | Red-Black |
|---------|-----|-----------|
| Balance strictness | Stricter (height diff ≤ 1) | Looser (≤ 2× height diff) |
| Search speed | Slightly faster | Slightly slower |
| Insert/Delete | More rotations | Fewer rotations |
| Best for | Read-heavy workloads | Write-heavy workloads |
| Used in | Databases | Language standard libraries |

---

## 3.5 Splay Tree 🟡

A self-adjusting BST that moves recently accessed elements to the root via **splaying**. No balance condition maintained.

```
After accessing node 3:
        8              3
       /              / \
      5       →      1   5
     /                  / \
    3                  4   8
   / \
  1   4

"Splay" = series of rotations to bring node to root
```

### Splay Operations

1. **Zig** — Single rotation (when parent is root)
2. **Zig-Zig** — Two same-direction rotations
3. **Zig-Zag** — Two opposite-direction rotations

**Amortized O(log n)** for all operations. Great for **temporal locality** (if you access something, you'll likely access it again).

**Used in:** Caches, garbage collectors, network routers.

---

## 3.6 Treap (Tree + Heap) 🟡

Each node has a **key** (BST property) and a **priority** (heap property, randomly assigned).

```
         (B, 99)         Key B, Priority 99
        /       \
    (A, 75)   (D, 80)
              /    \
          (C, 40)  (E, 50)

BST by key:    A < B < C < D < E  ✓
Heap by prio:  Each parent's priority > children's  ✓
```

Random priorities make the tree balanced with high probability → **expected O(log n)** for all operations.

---

## 3.7 B-Tree 🟡

A self-balancing tree designed for **disk-based storage** (databases, file systems). Each node can have **many children**.

### B-Tree of Order m

- Each node has at most **m children**
- Each non-root node has at least **⌈m/2⌉ children**
- Each node has at most **m-1 keys**
- All leaves are at the same level

```
B-Tree of order 3 (2-3 Tree):

           [16]
          /    \
      [7,13]   [20,25]
      / | \     / | \
    [1,4] [8,11] [14,15] [17,18] [21,23] [26,30]

Each node can hold 1-2 keys and have 2-3 children
```

### Why B-Trees?

- **Minimize disk reads** — wide nodes mean fewer levels
- **O(log n)** for all operations with small constant
- A B-tree with n keys and order m has height ≤ log_{m/2}(n)

```python
class BTreeNode:
    def __init__(self, leaf=True):
        self.keys = []
        self.children = []
        self.leaf = leaf

class BTree:
    def __init__(self, order):
        self.root = BTreeNode()
        self.order = order  # Max number of children
        self.max_keys = order - 1
        self.min_keys = (order + 1) // 2 - 1

    def search(self, node, key):
        i = 0
        while i < len(node.keys) and key > node.keys[i]:
            i += 1
        if i < len(node.keys) and key == node.keys[i]:
            return (node, i)
        if node.leaf:
            return None
        return self.search(node.children[i], key)

    def insert(self, key):
        root = self.root
        if len(root.keys) == self.max_keys:
            new_root = BTreeNode(leaf=False)
            new_root.children.append(self.root)
            self._split_child(new_root, 0)
            self.root = new_root
        self._insert_non_full(self.root, key)

    def _split_child(self, parent, index):
        order = self.order
        child = parent.children[index]
        mid = len(child.keys) // 2

        new_node = BTreeNode(leaf=child.leaf)
        parent.keys.insert(index, child.keys[mid])
        parent.children.insert(index + 1, new_node)

        new_node.keys = child.keys[mid+1:]
        child.keys = child.keys[:mid]

        if not child.leaf:
            new_node.children = child.children[mid+1:]
            child.children = child.children[:mid+1]

    def _insert_non_full(self, node, key):
        i = len(node.keys) - 1
        if node.leaf:
            node.keys.append(None)
            while i >= 0 and key < node.keys[i]:
                node.keys[i+1] = node.keys[i]
                i -= 1
            node.keys[i+1] = key
        else:
            while i >= 0 and key < node.keys[i]:
                i -= 1
            i += 1
            if len(node.children[i].keys) == self.max_keys:
                self._split_child(node, i)
                if key > node.keys[i]:
                    i += 1
            self._insert_non_full(node.children[i], key)
```

---

## 3.8 B+ Tree 🟡

A variation of B-Tree where:
- **All data is in leaf nodes** (internal nodes only store keys for routing)
- **Leaf nodes are linked** together (great for range queries)

```
Internal:    [15 | 25]
            /    |    \
Leaves:  [3,7,12] → [15,18,20] → [25,30,35]
         linked list of leaves for range scans
```

**Used in:** Nearly all databases (MySQL InnoDB, PostgreSQL) and file systems (NTFS, ext4).

---

## 3.9 B* Tree

Like a B-tree, but nodes must be at least **2/3 full** (instead of 1/2). This improves space utilization.

---

## 3.10 Trie (Prefix Tree) 🟢

A tree for storing strings where each edge represents a character. Shared prefixes share the same path.

```
Words: "cat", "car", "card", "do", "dog"

          (root)
         /      \
        c        d
        |        |
        a        o
       / \       |
      t   r      g
          |
          d

Path from root to any node = prefix
Nodes marked with * are end-of-word
```

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):  # O(m) where m = len(word)
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True

    def search(self, word):  # O(m)
        node = self.root
        for char in word:
            if char not in node.children:
                return False
            node = node.children[char]
        return node.is_end

    def starts_with(self, prefix):  # O(m)
        node = self.root
        for char in prefix:
            if char not in node.children:
                return False
            node = node.children[char]
        return True

    def autocomplete(self, prefix):
        """Return all words with given prefix"""
        node = self.root
        for char in prefix:
            if char not in node.children:
                return []
            node = node.children[char]

        results = []
        self._dfs(node, prefix, results)
        return results

    def _dfs(self, node, current, results):
        if node.is_end:
            results.append(current)
        for char, child in node.children.items():
            self._dfs(child, current + char, results)

# Usage
trie = Trie()
for word in ["apple", "app", "application", "apt", "banana"]:
    trie.insert(word)

print(trie.search("app"))          # True
print(trie.search("ap"))           # False
print(trie.starts_with("ap"))      # True
print(trie.autocomplete("app"))    # ['apple', 'app', 'application']
```

### Compressed Trie (Patricia Tree / Radix Tree)

Compresses chains of single-child nodes into one edge.

```
Standard Trie:          Compressed Trie:
   r                       r
   |                      / \
   o                   omane  ub
  / \                   |    / \
 m   b                  s   e   y
 |  / \                    |
 a e   y                   r
 |  \
 n   r
 |
 e
 |
 s

Words: "romanes", "ruber", "ruby"
```

---

## 3.11 Ternary Search Tree 🟡

Each node has three children: less-than, equal-to, and greater-than. More space-efficient than tries.

```python
class TSTNode:
    def __init__(self, char):
        self.char = char
        self.left = None    # Less than
        self.mid = None     # Equal to (continue matching)
        self.right = None   # Greater than
        self.is_end = False
```

---

## 3.12 Segment Tree 🟡

A tree for **range queries** and **point updates** on arrays. Each node stores aggregate information about a range.

```
Array: [2, 1, 5, 3, 4]

Segment tree for range sum:
              [0,4]=15
             /        \
        [0,2]=8      [3,4]=7
        /    \        /   \
    [0,1]=3  [2]=5  [3]=3 [4]=4
    /   \
  [0]=2 [1]=1

Each node stores the sum of its range
```

```python
class SegmentTree:
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self._build(arr, 1, 0, self.n - 1)

    def _build(self, arr, node, start, end):
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self._build(arr, 2 * node, start, mid)
            self._build(arr, 2 * node + 1, mid + 1, end)
            self.tree[node] = self.tree[2 * node] + self.tree[2 * node + 1]

    def query(self, l, r):
        """Range sum query [l, r] — O(log n)"""
        return self._query(1, 0, self.n - 1, l, r)

    def _query(self, node, start, end, l, r):
        if r < start or end < l:
            return 0  # Out of range
        if l <= start and end <= r:
            return self.tree[node]  # Completely in range
        mid = (start + end) // 2
        left_sum = self._query(2 * node, start, mid, l, r)
        right_sum = self._query(2 * node + 1, mid + 1, end, l, r)
        return left_sum + right_sum

    def update(self, idx, val):
        """Point update — O(log n)"""
        self._update(1, 0, self.n - 1, idx, val)

    def _update(self, node, start, end, idx, val):
        if start == end:
            self.tree[node] = val
        else:
            mid = (start + end) // 2
            if idx <= mid:
                self._update(2 * node, start, mid, idx, val)
            else:
                self._update(2 * node + 1, mid + 1, end, idx, val)
            self.tree[node] = self.tree[2 * node] + self.tree[2 * node + 1]

# Usage
arr = [2, 1, 5, 3, 4]
st = SegmentTree(arr)
print(st.query(1, 3))  # Sum of arr[1..3] = 1+5+3 = 9
st.update(2, 10)       # arr[2] = 10
print(st.query(1, 3))  # Sum of arr[1..3] = 1+10+3 = 14
```

### Lazy Propagation

For **range updates** (update all elements in a range), lazy propagation defers updates to children until needed → O(log n) per range update.

```python
class LazySegmentTree:
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self.lazy = [0] * (4 * self.n)
        self._build(arr, 1, 0, self.n - 1)

    def _build(self, arr, node, start, end):
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self._build(arr, 2 * node, start, mid)
            self._build(arr, 2 * node + 1, mid + 1, end)
            self.tree[node] = self.tree[2 * node] + self.tree[2 * node + 1]

    def _push_down(self, node, start, end):
        if self.lazy[node] != 0:
            mid = (start + end) // 2
            self._apply(2 * node, start, mid, self.lazy[node])
            self._apply(2 * node + 1, mid + 1, end, self.lazy[node])
            self.lazy[node] = 0

    def _apply(self, node, start, end, val):
        self.tree[node] += val * (end - start + 1)
        self.lazy[node] += val

    def range_update(self, l, r, val):
        """Add val to all elements in [l, r] — O(log n)"""
        self._range_update(1, 0, self.n - 1, l, r, val)

    def _range_update(self, node, start, end, l, r, val):
        if r < start or end < l:
            return
        if l <= start and end <= r:
            self._apply(node, start, end, val)
            return
        self._push_down(node, start, end)
        mid = (start + end) // 2
        self._range_update(2 * node, start, mid, l, r, val)
        self._range_update(2 * node + 1, mid + 1, end, l, r, val)
        self.tree[node] = self.tree[2 * node] + self.tree[2 * node + 1]
```

---

## 3.13 Fenwick Tree (Binary Indexed Tree / BIT) 🟡

A more compact alternative to segment trees for **prefix sum queries** and **point updates**.

```
Array:    [0, 1, 2, 3, 4, 5, 6, 7, 8]  (1-indexed)
BIT:      [0, 1, 3, 3, 10, 5, 11, 7, 36]

BIT[i] stores the sum of elements from (i - lowbit(i) + 1) to i
where lowbit(i) = i & (-i) = lowest set bit
```

```python
class FenwickTree:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)  # 1-indexed

    def update(self, i, delta):
        """Add delta to element at index i — O(log n)"""
        while i <= self.n:
            self.tree[i] += delta
            i += i & (-i)  # Move to parent

    def prefix_sum(self, i):
        """Sum of elements from index 1 to i — O(log n)"""
        total = 0
        while i > 0:
            total += self.tree[i]
            i -= i & (-i)  # Move to responsible ancestor
        return total

    def range_sum(self, l, r):
        """Sum of elements from index l to r — O(log n)"""
        return self.prefix_sum(r) - self.prefix_sum(l - 1)

# Usage
bit = FenwickTree(5)
for i, val in enumerate([2, 1, 5, 3, 4], 1):
    bit.update(i, val)

print(bit.range_sum(2, 4))  # 1 + 5 + 3 = 9
bit.update(3, 5)            # Add 5 to index 3
print(bit.range_sum(2, 4))  # 1 + 10 + 3 = 14
```

### 2D Fenwick Tree

For 2D range sum queries on a matrix:

```python
class FenwickTree2D:
    def __init__(self, rows, cols):
        self.rows = rows
        self.cols = cols
        self.tree = [[0] * (cols + 1) for _ in range(rows + 1)]

    def update(self, r, c, delta):
        i = r
        while i <= self.rows:
            j = c
            while j <= self.cols:
                self.tree[i][j] += delta
                j += j & (-j)
            i += i & (-i)

    def prefix_sum(self, r, c):
        total = 0
        i = r
        while i > 0:
            j = c
            while j > 0:
                total += self.tree[i][j]
                j -= j & (-j)
            i -= i & (-i)
        return total
```

---

## 3.14 Suffix Tree 🔴

A compressed trie of **all suffixes** of a string. Enables extremely fast string operations.

```
String: "banana$"
Suffixes: banana$, anana$, nana$, ana$, na$, a$, $

Compressed Suffix Tree:
          (root)
         / | \  \
        $  a  b  n
           |  |  |
          na  a  a
         / \  |  |
        $  na n  na
           |  a  |
           $  n  $
              a
              $
```

**Operations:** Pattern matching in O(m), longest repeated substring, longest common substring — all in linear time after O(n) construction (using Ukkonen's algorithm).

---

## 3.15 Suffix Array 🟡

A sorted array of all suffixes of a string. More space-efficient than suffix trees.

```
String: "banana"

Index  Suffix
  5    a
  3    ana
  1    anana
  0    banana
  4    na
  2    nana

Suffix Array: [5, 3, 1, 0, 4, 2]
```

```python
def build_suffix_array(s):
    """Simple O(n log²n) construction"""
    n = len(s)
    suffixes = [(s[i:], i) for i in range(n)]
    suffixes.sort()
    return [idx for _, idx in suffixes]

# With LCP (Longest Common Prefix) array
def build_lcp_array(s, sa):
    n = len(s)
    rank = [0] * n
    lcp = [0] * n
    for i in range(n):
        rank[sa[i]] = i

    k = 0
    for i in range(n):
        if rank[i] == 0:
            k = 0
            continue
        j = sa[rank[i] - 1]
        while i + k < n and j + k < n and s[i + k] == s[j + k]:
            k += 1
        lcp[rank[i]] = k
        if k > 0:
            k -= 1
    return lcp
```

---

## 3.16 K-D Tree (K-Dimensional Tree) 🟡

A BST generalized to multiple dimensions. Used for spatial data.

```
2D points: (7,2), (5,4), (9,6), (4,7), (8,1), (2,3)

         (7,2)          split on x
        /     \
    (5,4)    (9,6)      split on y
    /  \       /
 (2,3)(4,7) (8,1)      split on x
```

```python
class KDNode:
    def __init__(self, point, left=None, right=None, axis=0):
        self.point = point
        self.left = left
        self.right = right
        self.axis = axis

def build_kd_tree(points, depth=0):
    if not points:
        return None
    k = len(points[0])
    axis = depth % k
    points.sort(key=lambda p: p[axis])
    mid = len(points) // 2

    return KDNode(
        point=points[mid],
        left=build_kd_tree(points[:mid], depth + 1),
        right=build_kd_tree(points[mid + 1:], depth + 1),
        axis=axis
    )

def nearest_neighbor(root, target, best=None, best_dist=float('inf')):
    if root is None:
        return best, best_dist

    dist = sum((a - b) ** 2 for a, b in zip(root.point, target))
    if dist < best_dist:
        best, best_dist = root.point, dist

    axis = root.axis
    diff = target[axis] - root.point[axis]

    close = root.left if diff < 0 else root.right
    away = root.right if diff < 0 else root.left

    best, best_dist = nearest_neighbor(close, target, best, best_dist)

    if diff ** 2 < best_dist:
        best, best_dist = nearest_neighbor(away, target, best, best_dist)

    return best, best_dist
```

**Used in:** Nearest neighbor search, range search, computer graphics, machine learning (k-NN).

---

## 3.17 Interval Tree 🟡

Stores intervals and efficiently finds all intervals overlapping with a query point/interval.

```
Intervals: [15,20], [10,30], [17,19], [5,20], [12,15], [30,40]

                [15,20]
               /       \
         [10,30]      [17,19]
         /    \            \
    [5,20]  [12,15]     [30,40]
```

```python
class IntervalNode:
    def __init__(self, low, high):
        self.low = low
        self.high = high
        self.max_high = high  # Max high in this subtree
        self.left = None
        self.right = None

def insert_interval(root, low, high):
    if not root:
        return IntervalNode(low, high)
    if low < root.low:
        root.left = insert_interval(root.left, low, high)
    else:
        root.right = insert_interval(root.right, low, high)
    root.max_high = max(root.max_high, high)
    return root

def query_overlap(root, low, high):
    """Find all intervals overlapping with [low, high]"""
    results = []
    if not root:
        return results
    if root.low <= high and low <= root.high:
        results.append((root.low, root.high))
    if root.left and root.left.max_high >= low:
        results.extend(query_overlap(root.left, low, high))
    results.extend(query_overlap(root.right, low, high))
    return results
```

---

## 3.18 Merkle Tree 🟡

A tree of **hashes** — each leaf hashes data, and each internal node hashes its children. Used to verify data integrity.

```
          Hash(H12 + H34)        ← Root hash
         /                \
    Hash(H1 + H2)    Hash(H3 + H4)
      /       \         /       \
  Hash(D1) Hash(D2) Hash(D3) Hash(D4)  ← Leaf hashes
    D1       D2       D3       D4       ← Data blocks
```

**Used in:** Bitcoin/blockchain, Git, distributed systems, certificate transparency.

---

## 3.19 Cartesian Tree 🟡

A binary tree built from an array where:
- It satisfies the **heap property** (min or max)
- **Inorder traversal** gives the original array

```
Array: [3, 2, 6, 1, 9]

Min Cartesian Tree:
        1
       / \
      2   9
     / \
    3   6

Inorder: 3, 2, 6, 1, 9 ✓ (original array)
Heap: every parent ≤ children ✓
```

**Used in:** Range Minimum Query (RMQ), Treaps, building suffix trees.

---

## 3.20 Scapegoat Tree 🔴

A self-balancing BST that doesn't store extra balance information. When a tree becomes too unbalanced (a subtree is "heavy"), it **rebuilds** that subtree from scratch.

- **Weight-balanced**: a node is α-balanced if size(child) ≤ α × size(node)
- Typical α = 0.55 to 0.75
- Amortized O(log n) for all operations

---

## 3.21 Link-Cut Tree 🔴

A data structure for representing a **forest** (collection of trees) supporting:
- Link two trees by adding an edge
- Cut a tree into two by removing an edge
- Find root, path queries

Uses **splay trees** internally. Each operation runs in **amortized O(log n)**.

**Used in:** Dynamic graph connectivity, max flow algorithms.

---

## 3.22 Euler Tour Tree 🔴

Represents a tree using its Euler tour (sequence of nodes visited in DFS). Supports:
- Link/Cut operations
- Subtree queries (size, sum, etc.)
- Re-rooting

**Used in:** Dynamic tree problems, network connectivity.

---

## 3.23 Heavy-Light Decomposition (HLD) 🔴

Decomposes a tree into chains so that any path can be broken into O(log n) chains. Combined with segment trees, this gives O(log²n) path queries.

```
        1
       / \
      2   3         Heavy edges: 1→2, 2→4, 3→6
     / \   \        Light edges: 2→5, 1→3
    4   5   6

Chain 1: 1 → 2 → 4  (heavy path)
Chain 2: 5           (single node)
Chain 3: 3 → 6       (heavy path)
```

```python
class HLD:
    def __init__(self, n, adj):
        self.n = n
        self.adj = adj
        self.parent = [0] * n
        self.depth = [0] * n
        self.heavy = [-1] * n
        self.head = [0] * n
        self.pos = [0] * n
        self.size = [1] * n
        self.cur_pos = 0

        self._dfs_size(0, -1, 0)
        self._decompose(0, 0)

    def _dfs_size(self, v, p, d):
        self.parent[v] = p
        self.depth[v] = d
        max_size = 0
        for u in self.adj[v]:
            if u != p:
                self._dfs_size(u, v, d + 1)
                self.size[v] += self.size[u]
                if self.size[u] > max_size:
                    max_size = self.size[u]
                    self.heavy[v] = u

    def _decompose(self, v, h):
        self.head[v] = h
        self.pos[v] = self.cur_pos
        self.cur_pos += 1

        if self.heavy[v] != -1:
            self._decompose(self.heavy[v], h)

        for u in self.adj[v]:
            if u != self.parent[v] and u != self.heavy[v]:
                self._decompose(u, u)
```

---

## 3.24 Centroid Decomposition 🔴

Decomposes a tree by repeatedly finding centroids (nodes whose removal splits the tree into parts of size ≤ n/2). Creates a hierarchy of O(log n) levels.

**Used in:** Tree path queries, distance queries, competitive programming.

---

## 3.25 Van Emde Boas Tree 🔴

A tree for integers in the range [0, U) supporting operations in **O(log log U)** time.

| Operation | Time |
|-----------|------|
| Insert | O(log log U) |
| Delete | O(log log U) |
| Search | O(log log U) |
| Min/Max | O(1) |
| Successor/Predecessor | O(log log U) |

**Trade-off:** Uses O(U) space. Practical when U is not too large.

---

## 3.26 Wavelet Tree 🔴

A tree built on the alphabet of values in an array. Supports:
- Count of value v in range [l, r]: O(log σ)
- k-th smallest in range: O(log σ)
- Range frequency queries

Where σ = alphabet size.

---

## 3.27 Persistent Segment Tree 🔴

A segment tree that keeps **all historical versions** after updates. Each update creates a new root but shares most nodes with the previous version.

```
Version 0:  [sum=15]     Version 1:  [sum=20]  (after updating index 2)
            /     \                  /     \
         [8]     [7]             [13]     [7]   ← new node
         / \     / \             / \     / \
       [3] [5] [3] [4]        [3] [10] [3] [4]  ← new leaf
                                    ↑ updated

Shared nodes save memory: O(n + q log n) total space for q updates
```

```python
class PersistentNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class PersistentSegTree:
    def __init__(self, arr):
        self.n = len(arr)
        self.roots = [self._build(arr, 0, self.n - 1)]

    def _build(self, arr, l, r):
        if l == r:
            return PersistentNode(arr[l])
        mid = (l + r) // 2
        left = self._build(arr, l, mid)
        right = self._build(arr, mid + 1, r)
        return PersistentNode(left.val + right.val, left, right)

    def update(self, version, idx, val):
        new_root = self._update(self.roots[version], 0, self.n - 1, idx, val)
        self.roots.append(new_root)
        return len(self.roots) - 1

    def _update(self, node, l, r, idx, val):
        if l == r:
            return PersistentNode(val)
        mid = (l + r) // 2
        if idx <= mid:
            new_left = self._update(node.left, l, mid, idx, val)
            return PersistentNode(new_left.val + node.right.val, new_left, node.right)
        else:
            new_right = self._update(node.right, mid + 1, r, idx, val)
            return PersistentNode(node.left.val + new_right.val, node.left, new_right)
```

---

## 3.28 N-ary Tree / General Tree

A tree where each node can have **any number of children**.

```python
class NaryNode:
    def __init__(self, val):
        self.val = val
        self.children = []

# Example: File system
root = NaryNode("/")
root.children.append(NaryNode("home"))
root.children.append(NaryNode("etc"))
root.children[0].children.append(NaryNode("user"))
```

---

## 3.29 Rope (String Tree) 🟡

A balanced binary tree for efficiently manipulating long strings. Each leaf holds a short string, and internal nodes store the total length of their left subtree.

```
        [11]            weight = length of left subtree
       /    \
    "Hello "  [5]
             /   \
         "my "  "name"

Concatenated: "Hello my name"
```

**Operations:**
- Concatenation: O(log n)
- Split: O(log n)
- Insert: O(log n)
- Delete: O(log n)

**Used in:** Text editors (xi editor), large text processing.

---

## 3.30 Order-Statistic Tree 🟡

BST augmented with subtree sizes for O(log n) rank queries.

```python
class OSNode:
    def __init__(self, key):
        self.key = key
        self.left = self.right = None
        self.size = 1  # Subtree size including self

def get_size(node):
    return node.size if node else 0

def update_size(node):
    if node:
        node.size = 1 + get_size(node.left) + get_size(node.right)

def os_select(root, k):
    """Find k-th smallest element (1-indexed)"""
    left_size = get_size(root.left)
    if k == left_size + 1:
        return root.key
    elif k <= left_size:
        return os_select(root.left, k)
    else:
        return os_select(root.right, k - left_size - 1)

def os_rank(root, key):
    """Find rank of key (how many elements are smaller)"""
    if root is None:
        return 0
    if key < root.key:
        return os_rank(root.left, key)
    elif key > root.key:
        return 1 + get_size(root.left) + os_rank(root.right, key)
    else:
        return get_size(root.left) + 1
```

> **Augmented BST pattern:** Store extra info per node (subtree size, sum, max, min, etc.) and maintain during rotations. Any balanced BST (AVL, Red-Black) can be augmented.

---

## 3.31 Quadtree / Octree 🟡

Recursively partition 2D/3D space into quadrants/octants.

```python
class QuadTree:
    def __init__(self, x, y, w, h, capacity=4):
        self.boundary = (x, y, w, h)  # center x,y and half-width/height
        self.capacity = capacity
        self.points = []
        self.divided = False
        self.nw = self.ne = self.sw = self.se = None

    def contains(self, px, py):
        x, y, w, h = self.boundary
        return x - w <= px < x + w and y - h <= py < y + h

    def subdivide(self):
        x, y, w, h = self.boundary
        hw, hh = w / 2, h / 2
        self.nw = QuadTree(x - hw, y - hh, hw, hh, self.capacity)
        self.ne = QuadTree(x + hw, y - hh, hw, hh, self.capacity)
        self.sw = QuadTree(x - hw, y + hh, hw, hh, self.capacity)
        self.se = QuadTree(x + hw, y + hh, hw, hh, self.capacity)
        self.divided = True

    def insert(self, px, py):
        if not self.contains(px, py):
            return False
        if len(self.points) < self.capacity:
            self.points.append((px, py))
            return True
        if not self.divided:
            self.subdivide()
        return (self.nw.insert(px, py) or self.ne.insert(px, py) or
                self.sw.insert(px, py) or self.se.insert(px, py))

# Used in: image compression, collision detection, GIS, Barnes-Hut simulation
# Octree: 3D version with 8 children — used in 3D graphics and physics
```

---

## 3.32 R-Tree 🟡

Balanced tree for spatial indexing of rectangles/polygons. Used in PostGIS, SQLite.

```
       ┌──── R1 ────┐
       │             │
   ┌── R3 ──┐   ┌── R4 ──┐
   │        │   │        │
  obj1   obj2  obj3   obj4

Each node stores a bounding rectangle containing all children.
Search: prune branches whose bounding box doesn't overlap query.
Insert: choose subtree with minimum bounding box enlargement.
```

---

## 3.33 AA Tree 🟡

Simplified Red-Black tree — only right children can be red.

```
Rules:
1. Every node has a level (leaves are level 1)
2. Left child level = parent level - 1
3. Right child level = parent level or parent level - 1
4. Right grandchild level < grandparent level

Only two operations needed: skew (right rotation) and split (left rotation)
```

---

## 3.34 2-3-4 Tree 🟡

B-tree of order 4. Each node has 2, 3, or 4 children. **Isomorphic to Red-Black tree** — understanding one gives you the other.

```
2-3-4 Node     ↔    Red-Black equivalent
  [a]          ↔    black(a)
  [a,b]        ↔    black(b) with red left child(a)
  [a,b,c]      ↔    black(b) with red children(a,c)
```

---

## 3.35 Ball Tree / VP-Tree / Cover Tree 🔴

Metric space trees for nearest neighbor search in **arbitrary distance metrics** (not just Euclidean).

```python
# VP-Tree (Vantage Point Tree)
class VPNode:
    def __init__(self, point, radius, left, right):
        self.point = point
        self.radius = radius  # Median distance from vantage point
        self.left = left      # Points within radius
        self.right = right    # Points outside radius

# Ball Tree: each node is a hypersphere containing its subtree
# Cover Tree: maintains covering invariant at each level
# Used in: ML (k-NN with non-Euclidean metrics), protein structure search
```

---

## 3.36 Range Tree 🔴

Multi-level tree for orthogonal range queries in d dimensions.

```
Query: find all points in [x1,x2] × [y1,y2]
1D: balanced BST → O(log n + k)
2D: BST on x → each node has BST on y for its subtree → O(log²n + k)
With fractional cascading: O(log n + k) for 2D
```

---

## Summary Comparison

| Tree | Time Complexity | Space | Best For |
|------|----------------|-------|----------|
| Binary Tree | O(n) worst | O(n) | Hierarchical data |
| BST | O(log n) avg, O(n) worst | O(n) | Ordered data |
| AVL | O(log n) guaranteed | O(n) | Read-heavy workloads |
| Red-Black | O(log n) guaranteed | O(n) | Write-heavy workloads |
| B-Tree | O(log n) w/ small constant | O(n) | Disk-based storage |
| Trie | O(m) per operation | O(n·m) | String prefix operations |
| Segment Tree | O(log n) query/update | O(n) | Range queries |
| Fenwick Tree | O(log n) query/update | O(n) | Prefix sums |
| K-D Tree | O(log n) avg | O(n) | Multi-dimensional queries |
| Splay | O(log n) amortized | O(n) | Temporal locality |
| Order-Statistic | O(log n) | O(n) | Rank queries |
| Quadtree | O(log n) avg | O(n) | 2D spatial partitioning |
| R-Tree | O(log n) | O(n) | Rectangle/polygon indexing |

---

[← Previous: Linear Data Structures](02-data-structures-linear.md) | [Next: Hash Tables, Heaps & Graphs →](04-data-structures-hash-heap-graph.md)
