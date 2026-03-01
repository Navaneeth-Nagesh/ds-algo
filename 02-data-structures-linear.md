# Chapter 2: Linear Data Structures 🟢

## Overview

Linear data structures store elements in a sequential order. They are the building blocks of all programming and the first structures you should master.

---

## 2.1 Arrays

### What is an Array?

An array is a **contiguous block of memory** storing elements of the same type, accessible by index.

```
Index:   0     1     2     3     4
       ┌─────┬─────┬─────┬─────┬─────┐
       │  10 │  20 │  30 │  40 │  50 │
       └─────┴─────┴─────┴─────┴─────┘
       Memory addresses are consecutive
```

### Why Arrays are Fast for Access

Each element is at a **predictable memory address**:
```
address(arr[i]) = base_address + (i × element_size)
```
This makes access O(1) — no matter which index, one calculation gets you there.

### Operations & Complexity

| Operation | Time | Why |
|-----------|------|-----|
| Access by index | O(1) | Direct address calculation |
| Search (unsorted) | O(n) | Must check each element |
| Search (sorted) | O(log n) | Binary search |
| Insert at end | O(1)* | Amortized for dynamic arrays |
| Insert at beginning | O(n) | Must shift all elements right |
| Insert at middle | O(n) | Must shift elements |
| Delete at end | O(1) | Just decrement size |
| Delete at beginning | O(n) | Must shift all elements left |
| Delete at middle | O(n) | Must shift elements |

### Static vs Dynamic Arrays

```python
# ===== STATIC ARRAY (fixed size) =====
# In C: int arr[5] = {1, 2, 3, 4, 5};
# In Python, we can simulate with:
import array
static_arr = array.array('i', [1, 2, 3, 4, 5])

# ===== DYNAMIC ARRAY (resizable) =====
# Python lists are dynamic arrays
dynamic_arr = [1, 2, 3, 4, 5]
dynamic_arr.append(6)  # Automatically resizes if needed

# How dynamic arrays work internally:
# 1. Start with capacity 4
# 2. When full, allocate new array of size 8 (double)
# 3. Copy all elements to new array
# 4. Delete old array
# This gives amortized O(1) for append
```

### Multi-Dimensional Arrays

```python
# 2D Array (Matrix) — used for grids, images, graphs
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]
# Access: matrix[row][col]
print(matrix[1][2])  # Output: 6

# 3D Array — used for 3D grids, video frames
cube = [[[0 for _ in range(3)] for _ in range(3)] for _ in range(3)]

# Jagged Array — rows have different lengths
jagged = [
    [1, 2],
    [3, 4, 5, 6],
    [7]
]
```

### Common Array Techniques

```python
# ---- Two Pointer Technique ----
# Find if a sorted array has a pair that sums to target
def two_sum_sorted(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        current_sum = arr[left] + arr[right]
        if current_sum == target:
            return (left, right)
        elif current_sum < target:
            left += 1
        else:
            right -= 1
    return None

# ---- Sliding Window ----
# Find maximum sum subarray of size k
def max_sum_subarray(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]  # Slide: add new, remove old
        max_sum = max(max_sum, window_sum)
    return max_sum

# ---- Prefix Sum ----
# Answer range sum queries in O(1) after O(n) preprocessing
def build_prefix_sum(arr):
    prefix = [0] * (len(arr) + 1)
    for i in range(len(arr)):
        prefix[i + 1] = prefix[i] + arr[i]
    return prefix

def range_sum(prefix, left, right):  # inclusive
    return prefix[right + 1] - prefix[left]

# ---- Kadane's Algorithm ----
# Find maximum sum contiguous subarray in O(n)
def max_subarray_sum(arr):
    max_ending_here = max_so_far = arr[0]
    for num in arr[1:]:
        max_ending_here = max(num, max_ending_here + num)
        max_so_far = max(max_so_far, max_ending_here)
    return max_so_far
```

### When to Use Arrays

- ✅ Need fast random access by index
- ✅ Data size is known or changes infrequently
- ✅ Cache-friendly sequential access needed
- ❌ Frequent insertions/deletions in the middle
- ❌ Size changes dramatically and unpredictably

---

## 2.2 Strings

### What is a String?

A string is a **sequence of characters**. Internally, it's usually an array of characters (or bytes).

```python
# Strings in Python are immutable sequences
s = "hello"
# s[0] = 'H'  # ERROR: strings are immutable in Python

# Common string operations
s = "Hello, World!"
print(len(s))           # 13
print(s[0])             # 'H'
print(s[7:12])          # 'World'  (slicing)
print(s.lower())        # 'hello, world!'
print(s.find("World"))  # 7
print(s.replace("World", "Python"))  # 'Hello, Python!'
print(s.split(", "))    # ['Hello', 'World!']
print("".join(['a','b','c']))  # 'abc'
```

### String Complexity Gotchas

```python
# BAD: O(n²) — string concatenation in a loop creates new string each time
result = ""
for i in range(n):
    result += str(i)  # Each += creates a new string object

# GOOD: O(n) — use a list and join
parts = []
for i in range(n):
    parts.append(str(i))
result = "".join(parts)
```

### String Representations

| Encoding | Bytes per char | Use case |
|----------|---------------|----------|
| ASCII | 1 | English text (128 chars) |
| UTF-8 | 1-4 | Universal standard (web) |
| UTF-16 | 2-4 | Windows, Java, JavaScript |
| UTF-32 | 4 | Fixed-width Unicode |

### When to Use Strings

- ✅ Text processing, parsing, pattern matching
- ✅ Keys in hash maps
- ❌ Frequent character-level modifications (use list of chars or StringBuilder)

---

## 2.3 Linked Lists

### What is a Linked List?

A linked list stores elements in **nodes**, where each node points to the next. Elements are NOT contiguous in memory.

```
Singly Linked List:
  ┌────┬───┐    ┌────┬───┐    ┌────┬───┐    ┌────┬──────┐
  │ 10 │ ──┼───→│ 20 │ ──┼───→│ 30 │ ──┼───→│ 40 │ None │
  └────┴───┘    └────┴───┘    └────┴───┘    └────┴──────┘
  head                                        tail
```

### Singly Linked List Implementation

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class SinglyLinkedList:
    def __init__(self):
        self.head = None

    def is_empty(self):
        return self.head is None

    # Insert at the beginning — O(1)
    def insert_at_head(self, data):
        new_node = Node(data)
        new_node.next = self.head
        self.head = new_node

    # Insert at the end — O(n) [O(1) if we maintain tail pointer]
    def insert_at_tail(self, data):
        new_node = Node(data)
        if not self.head:
            self.head = new_node
            return
        current = self.head
        while current.next:
            current = current.next
        current.next = new_node

    # Delete a node with given value — O(n)
    def delete(self, data):
        if not self.head:
            return
        if self.head.data == data:
            self.head = self.head.next
            return
        current = self.head
        while current.next:
            if current.next.data == data:
                current.next = current.next.next
                return
            current = current.next

    # Search — O(n)
    def search(self, data):
        current = self.head
        while current:
            if current.data == data:
                return True
            current = current.next
        return False

    # Reverse — O(n) time, O(1) space
    def reverse(self):
        prev = None
        current = self.head
        while current:
            next_node = current.next
            current.next = prev
            prev = current
            current = next_node
        self.head = prev

    # Print the list
    def display(self):
        elements = []
        current = self.head
        while current:
            elements.append(str(current.data))
            current = current.next
        print(" → ".join(elements) + " → None")

# Usage
ll = SinglyLinkedList()
ll.insert_at_head(30)
ll.insert_at_head(20)
ll.insert_at_head(10)
ll.insert_at_tail(40)
ll.display()  # 10 → 20 → 30 → 40 → None
ll.reverse()
ll.display()  # 40 → 30 → 20 → 10 → None
```

### Doubly Linked List

Each node has pointers to BOTH the next and previous nodes.

```
  None ←──┬────┬──→ ←──┬────┬──→ ←──┬────┬──→ None
          │ 10 │       │ 20 │       │ 30 │
          └────┘       └────┘       └────┘
          head                       tail
```

```python
class DoublyNode:
    def __init__(self, data):
        self.data = data
        self.prev = None
        self.next = None

class DoublyLinkedList:
    def __init__(self):
        self.head = None
        self.tail = None

    def insert_at_head(self, data):
        new_node = DoublyNode(data)
        if not self.head:
            self.head = self.tail = new_node
        else:
            new_node.next = self.head
            self.head.prev = new_node
            self.head = new_node

    def insert_at_tail(self, data):
        new_node = DoublyNode(data)
        if not self.tail:
            self.head = self.tail = new_node
        else:
            new_node.prev = self.tail
            self.tail.next = new_node
            self.tail = new_node

    def delete(self, data):
        current = self.head
        while current:
            if current.data == data:
                if current.prev:
                    current.prev.next = current.next
                else:
                    self.head = current.next
                if current.next:
                    current.next.prev = current.prev
                else:
                    self.tail = current.prev
                return
            current = current.next

    def display_forward(self):
        elements = []
        current = self.head
        while current:
            elements.append(str(current.data))
            current = current.next
        print("None ← " + " ↔ ".join(elements) + " → None")
```

### Circular Linked List

The last node points back to the first node.

```
  ┌────┬───┐    ┌────┬───┐    ┌────┬───┐
  │ 10 │ ──┼───→│ 20 │ ──┼───→│ 30 │ ──┼───┐
  └────┴───┘    └────┴───┘    └────┴───┘    │
    ↑                                        │
    └────────────────────────────────────────┘
```

```python
class CircularLinkedList:
    def __init__(self):
        self.head = None

    def insert(self, data):
        new_node = Node(data)
        if not self.head:
            self.head = new_node
            new_node.next = self.head  # Points to itself
        else:
            current = self.head
            while current.next != self.head:
                current = current.next
            current.next = new_node
            new_node.next = self.head

    def display(self):
        if not self.head:
            return
        elements = []
        current = self.head
        while True:
            elements.append(str(current.data))
            current = current.next
            if current == self.head:
                break
        print(" → ".join(elements) + " → (back to head)")
```

### XOR Linked List (Memory-Efficient Doubly Linked List)

Instead of storing `prev` and `next`, store `prev XOR next`. This halves pointer storage.

```
Node stores: address(prev) XOR address(next)

To traverse forward:  next = node.xor_ptr XOR prev_address
To traverse backward: prev = node.xor_ptr XOR next_address
```

This is a memory optimization used in embedded systems. Python doesn't support this natively (no raw pointers), but it's important in C/C++.

### Unrolled Linked List

Each node stores an **array of elements** instead of a single element. Combines benefits of arrays (cache-friendly) and linked lists (efficient insertion).

```
  ┌──────────────┬───┐    ┌──────────────┬───┐
  │ [1, 2, 3, 4] │ ──┼───→│ [5, 6, 7, _] │ ──┼───→ None
  └──────────────┴───┘    └──────────────┴───┘
```

### Comparison: Array vs Linked List

| Feature | Array | Linked List |
|---------|-------|-------------|
| Access by index | O(1) ✅ | O(n) ❌ |
| Insert at beginning | O(n) ❌ | O(1) ✅ |
| Insert at end | O(1)* ✅ | O(1) with tail ✅ |
| Insert in middle | O(n) | O(1) after finding position |
| Delete | O(n) | O(1) after finding position |
| Memory | Contiguous, cache-friendly | Scattered, pointer overhead |
| Size | Fixed or amortized resize | Truly dynamic |

---

## 2.4 Stacks

### What is a Stack?

A stack follows **LIFO** (Last In, First Out) — like a stack of plates.

```
        ┌─────┐
        │  40 │  ← top (last in, first out)
        ├─────┤
        │  30 │
        ├─────┤
        │  20 │
        ├─────┤
        │  10 │
        └─────┘

  push(50):          pop():
        ┌─────┐           ┌─────┐
        │  50 │ ← new     │     │ ← 40 removed
        ├─────┤           ├─────┤
        │  40 │           │  30 │ ← new top
        ├─────┤           ├─────┤
        │  30 │           │  20 │
        ├─────┤           ├─────┤
        │  20 │           │  10 │
        ├─────┤           └─────┘
        │  10 │
        └─────┘
```

### Operations & Complexity

| Operation | Time | Description |
|-----------|------|-------------|
| push(x) | O(1) | Add element to top |
| pop() | O(1) | Remove and return top element |
| peek/top() | O(1) | View top element without removing |
| isEmpty() | O(1) | Check if stack is empty |
| size() | O(1) | Number of elements |

### Implementation

```python
# ===== Using Python list (most common) =====
class Stack:
    def __init__(self):
        self.items = []

    def push(self, item):
        self.items.append(item)

    def pop(self):
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self.items.pop()

    def peek(self):
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self.items[-1]

    def is_empty(self):
        return len(self.items) == 0

    def size(self):
        return len(self.items)

# ===== Using Linked List =====
class StackLL:
    def __init__(self):
        self.head = None
        self._size = 0

    def push(self, data):
        new_node = Node(data)
        new_node.next = self.head
        self.head = new_node
        self._size += 1

    def pop(self):
        if not self.head:
            raise IndexError("Stack is empty")
        data = self.head.data
        self.head = self.head.next
        self._size -= 1
        return data
```

### Classic Stack Problems

```python
# ---- 1. Balanced Parentheses ----
def is_balanced(expression):
    stack = []
    matching = {')': '(', ']': '[', '}': '{'}
    for char in expression:
        if char in '([{':
            stack.append(char)
        elif char in ')]}':
            if not stack or stack[-1] != matching[char]:
                return False
            stack.pop()
    return len(stack) == 0

print(is_balanced("({[]})"))  # True
print(is_balanced("({[})"))   # False

# ---- 2. Evaluate Postfix Expression ----
def eval_postfix(expression):
    stack = []
    for token in expression.split():
        if token.isdigit():
            stack.append(int(token))
        else:
            b, a = stack.pop(), stack.pop()
            if token == '+': stack.append(a + b)
            elif token == '-': stack.append(a - b)
            elif token == '*': stack.append(a * b)
            elif token == '/': stack.append(int(a / b))
    return stack[0]

print(eval_postfix("3 4 + 2 *"))  # (3 + 4) * 2 = 14

# ---- 3. Next Greater Element ----
def next_greater(arr):
    result = [-1] * len(arr)
    stack = []  # stores indices
    for i in range(len(arr)):
        while stack and arr[i] > arr[stack[-1]]:
            result[stack.pop()] = arr[i]
        stack.append(i)
    return result

print(next_greater([4, 5, 2, 10, 8]))  # [5, 10, 10, -1, -1]

# ---- 4. Min Stack (get minimum in O(1)) ----
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []

    def push(self, val):
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)

    def pop(self):
        val = self.stack.pop()
        if val == self.min_stack[-1]:
            self.min_stack.pop()
        return val

    def get_min(self):
        return self.min_stack[-1]

# ---- 5. Infix to Postfix (Shunting Yard Algorithm) ----
def infix_to_postfix(expression):
    precedence = {'+': 1, '-': 1, '*': 2, '/': 2, '^': 3}
    right_assoc = {'^'}
    output = []
    stack = []

    for token in expression.split():
        if token.isalnum():
            output.append(token)
        elif token == '(':
            stack.append(token)
        elif token == ')':
            while stack and stack[-1] != '(':
                output.append(stack.pop())
            stack.pop()  # Remove '('
        else:  # Operator
            while (stack and stack[-1] != '(' and
                   stack[-1] in precedence and
                   (precedence[stack[-1]] > precedence[token] or
                    (precedence[stack[-1]] == precedence[token] and token not in right_assoc))):
                output.append(stack.pop())
            stack.append(token)

    while stack:
        output.append(stack.pop())
    return " ".join(output)
```

### Real-World Uses of Stacks

- Function call stack (recursion)
- Undo/Redo operations
- Browser back/forward buttons
- Expression parsing and evaluation
- Depth-First Search (DFS)
- Syntax checking (compilers)

---

## 2.5 Queues

### What is a Queue?

A queue follows **FIFO** (First In, First Out) — like a line at a store.

```
  Enqueue →  ┌────┬────┬────┬────┐  → Dequeue
             │ 40 │ 30 │ 20 │ 10 │
             └────┴────┴────┴────┘
             rear              front
```

### Operations & Complexity

| Operation | Time | Description |
|-----------|------|-------------|
| enqueue(x) | O(1) | Add to rear |
| dequeue() | O(1) | Remove from front |
| front/peek() | O(1) | View front element |
| isEmpty() | O(1) | Check if empty |

### Queue Implementations

```python
# ===== 1. Simple Queue using list (NOT recommended — dequeue is O(n)) =====
class SimpleQueue:
    def __init__(self):
        self.items = []

    def enqueue(self, item):
        self.items.append(item)      # O(1)

    def dequeue(self):
        return self.items.pop(0)     # O(n) — BAD!

# ===== 2. Queue using collections.deque (RECOMMENDED) =====
from collections import deque

class Queue:
    def __init__(self):
        self.items = deque()

    def enqueue(self, item):
        self.items.append(item)      # O(1)

    def dequeue(self):
        return self.items.popleft()  # O(1) ✓

    def peek(self):
        return self.items[0]

    def is_empty(self):
        return len(self.items) == 0

    def size(self):
        return len(self.items)

# ===== 3. Queue using Linked List =====
class QueueLL:
    def __init__(self):
        self.head = None  # front
        self.tail = None  # rear

    def enqueue(self, data):
        new_node = Node(data)
        if self.tail:
            self.tail.next = new_node
        self.tail = new_node
        if not self.head:
            self.head = new_node

    def dequeue(self):
        if not self.head:
            raise IndexError("Queue is empty")
        data = self.head.data
        self.head = self.head.next
        if not self.head:
            self.tail = None
        return data
```

### Circular Queue (Ring Buffer)

Uses a fixed-size array with wrap-around. Efficient for streaming/buffering.

```
  Capacity: 5
  ┌────┬────┬────┬────┬────┐
  │  _ │ 20 │ 30 │ 40 │  _ │
  └────┴────┴────┴────┴────┘
    ↑    ↑              ↑
   (4)  front(1)      rear(3)

  After enqueue(50):
  ┌────┬────┬────┬────┬────┐
  │  _ │ 20 │ 30 │ 40 │ 50 │
  └────┴────┴────┴────┴────┘
         ↑              ↑
       front(1)       rear(4)

  After enqueue(60): wraps around!
  ┌────┬────┬────┬────┬────┐
  │ 60 │ 20 │ 30 │ 40 │ 50 │
  └────┴────┴────┴────┴────┘
    ↑    ↑
  rear  front
```

```python
class CircularQueue:
    def __init__(self, capacity):
        self.queue = [None] * capacity
        self.capacity = capacity
        self.front = 0
        self.rear = -1
        self.size = 0

    def enqueue(self, item):
        if self.size == self.capacity:
            raise OverflowError("Queue is full")
        self.rear = (self.rear + 1) % self.capacity
        self.queue[self.rear] = item
        self.size += 1

    def dequeue(self):
        if self.size == 0:
            raise IndexError("Queue is empty")
        item = self.queue[self.front]
        self.front = (self.front + 1) % self.capacity
        self.size -= 1
        return item

    def peek(self):
        if self.size == 0:
            raise IndexError("Queue is empty")
        return self.queue[self.front]

    def is_full(self):
        return self.size == self.capacity
```

---

## 2.6 Deque (Double-Ended Queue)

A deque allows insertion and deletion at **both ends**.

```
  ┌────┬────┬────┬────┐
  │ 10 │ 20 │ 30 │ 40 │
  └────┴────┴────┴────┘
  ↕ front              ↕ rear
  Can add/remove        Can add/remove
  from this end         from this end
```

```python
from collections import deque

dq = deque()

# Operations — all O(1)
dq.append(10)       # Add to right:  [10]
dq.append(20)       # Add to right:  [10, 20]
dq.appendleft(5)    # Add to left:   [5, 10, 20]
dq.pop()            # Remove right:  [5, 10]     returns 20
dq.popleft()        # Remove left:   [10]        returns 5

# Deque as sliding window maximum
def max_sliding_window(nums, k):
    """Find maximum in each window of size k — O(n) total"""
    dq = deque()  # Store indices of potential maximums
    result = []

    for i in range(len(nums)):
        # Remove elements outside window
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # Remove smaller elements (they'll never be max)
        while dq and nums[dq[-1]] < nums[i]:
            dq.pop()

        dq.append(i)

        if i >= k - 1:
            result.append(nums[dq[0]])

    return result

print(max_sliding_window([1, 3, -1, -3, 5, 3, 6, 7], 3))
# Output: [3, 3, 5, 5, 6, 7]
```

---

## 2.7 Priority Queue

Elements have **priorities**; the highest priority element is dequeued first. (Covered in detail in the Heaps chapter, but introduced here.)

```python
import heapq

# Python's heapq implements a min-heap (smallest = highest priority)
pq = []
heapq.heappush(pq, (3, "Low priority"))
heapq.heappush(pq, (1, "High priority"))
heapq.heappush(pq, (2, "Medium priority"))

while pq:
    priority, item = heapq.heappop(pq)
    print(f"Priority {priority}: {item}")
# Output:
# Priority 1: High priority
# Priority 2: Medium priority
# Priority 3: Low priority

# For max-heap, negate priorities
heapq.heappush(pq, (-3, "High priority"))
heapq.heappush(pq, (-1, "Low priority"))
```

---

## 2.8 Monotonic Stack & Monotonic Queue

### Monotonic Stack

A stack where elements are always in sorted order (either increasing or decreasing).

```python
# Monotonic Decreasing Stack — useful for Next Greater Element
def next_greater_element(arr):
    n = len(arr)
    result = [-1] * n
    stack = []  # Decreasing stack (stores indices)

    for i in range(n):
        # Pop elements smaller than current
        while stack and arr[i] > arr[stack[-1]]:
            idx = stack.pop()
            result[idx] = arr[i]
        stack.append(i)

    return result

# Largest Rectangle in Histogram
def largest_rectangle(heights):
    stack = []  # Increasing stack (stores indices)
    max_area = 0

    for i, h in enumerate(heights + [0]):  # Append 0 to flush stack
        while stack and heights[stack[-1]] > h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    return max_area

print(largest_rectangle([2, 1, 5, 6, 2, 3]))  # 10
```

### Monotonic Queue

A deque where elements are maintained in sorted order — used for sliding window min/max problems.

```python
# Already shown above in the sliding window maximum example
# The deque maintains a decreasing order of values
```

---

## 2.9 Circular Buffer (Ring Buffer)

A fixed-size buffer that wraps around. Used extensively in:
- Audio/video streaming
- Network packet buffering
- Producer-consumer problems
- Keyboard input buffers

```python
class RingBuffer:
    def __init__(self, capacity):
        self.buffer = [None] * capacity
        self.capacity = capacity
        self.head = 0       # Read position
        self.tail = 0       # Write position
        self.count = 0

    def write(self, data):
        if self.count == self.capacity:
            # Overwrite oldest data
            self.head = (self.head + 1) % self.capacity
        else:
            self.count += 1
        self.buffer[self.tail] = data
        self.tail = (self.tail + 1) % self.capacity

    def read(self):
        if self.count == 0:
            raise BufferError("Buffer is empty")
        data = self.buffer[self.head]
        self.head = (self.head + 1) % self.capacity
        self.count -= 1
        return data
```

---

## 2.10 Gap Buffer

Used in text editors for efficient insertion at the cursor position.

```
"Hello, World!" with cursor at position 7:

  H  e  l  l  o  ,     [___GAP___]  W  o  r  l  d  !

Insert 'Beautiful ' at cursor:
  H  e  l  l  o  ,     B  e  a  u  t  i  f  u  l     [_GAP_]  W  o  r  l  d  !
```

```python
class GapBuffer:
    def __init__(self, initial_gap=10):
        self.buffer = [None] * initial_gap
        self.gap_start = 0
        self.gap_end = initial_gap

    def move_cursor(self, position):
        while self.gap_start < position:
            self.buffer[self.gap_start] = self.buffer[self.gap_end]
            self.gap_start += 1
            self.gap_end += 1
        while self.gap_start > position:
            self.gap_start -= 1
            self.gap_end -= 1
            self.buffer[self.gap_end] = self.buffer[self.gap_start]

    def insert(self, char):
        if self.gap_start == self.gap_end:
            self._expand_gap()
        self.buffer[self.gap_start] = char
        self.gap_start += 1

    def delete(self):
        if self.gap_start > 0:
            self.gap_start -= 1
```

---

## 2.10 Difference Array 🟢

Apply range updates in O(1) each, then reconstruct with prefix sum.

```python
class DifferenceArray:
    """Range update [l, r] += val in O(1), query all values in O(n)"""
    def __init__(self, arr):
        self.n = len(arr)
        self.diff = [0] * (self.n + 1)
        self.diff[0] = arr[0]
        for i in range(1, self.n):
            self.diff[i] = arr[i] - arr[i-1]

    def range_update(self, l, r, val):
        self.diff[l] += val
        if r + 1 < self.n:
            self.diff[r + 1] -= val

    def build(self):
        result = [0] * self.n
        result[0] = self.diff[0]
        for i in range(1, self.n):
            result[i] = result[i-1] + self.diff[i]
        return result

# Complement to prefix sums:
# Prefix sum: O(n) build, O(1) range query
# Difference array: O(1) range update, O(n) rebuild
```

---

## 2.11 Bit Array (Bitvector) 🟢

Pack booleans into individual bits — 8× memory savings.

```python
class BitArray:
    def __init__(self, size):
        self.size = size
        self.data = bytearray((size + 7) // 8)

    def set(self, i):
        self.data[i >> 3] |= (1 << (i & 7))

    def clear(self, i):
        self.data[i >> 3] &= ~(1 << (i & 7))

    def get(self, i):
        return bool(self.data[i >> 3] & (1 << (i & 7)))

    def count_ones(self):
        return sum(bin(b).count('1') for b in self.data)

# Foundation for succinct data structures (rank/select operations)
```

---

## 2.12 Sparse Array 🟢

Store only non-zero elements — critical for scientific computing and NLP.

```python
class SparseArray:
    """Stores only non-default values"""
    def __init__(self, size, default=0):
        self.size = size
        self.default = default
        self.data = {}  # index → value

    def get(self, i):
        return self.data.get(i, self.default)

    def set(self, i, val):
        if val == self.default:
            self.data.pop(i, None)
        else:
            self.data[i] = val

    def nnz(self):
        return len(self.data)
```

---

## 2.13 Sliding Window & Two-Pointer Templates 🟢

### Fixed-Size Sliding Window

```python
def fixed_window(arr, k):
    """Template: window of exactly size k."""
    window_sum = sum(arr[:k])
    best = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]
        best = max(best, window_sum)
    return best
```

### Variable-Size Sliding Window

```python
def variable_window(s):
    """Template: longest substring without repeating characters."""
    seen = {}
    left = 0
    best = 0
    for right, ch in enumerate(s):
        if ch in seen and seen[ch] >= left:
            left = seen[ch] + 1
        seen[ch] = right
        best = max(best, right - left + 1)
    return best
```

### Two Pointers: Opposite Direction

```python
def two_sum_sorted(arr, target):
    """Two pointers from both ends."""
    lo, hi = 0, len(arr) - 1
    while lo < hi:
        s = arr[lo] + arr[hi]
        if s == target:
            return [lo, hi]
        elif s < target:
            lo += 1
        else:
            hi -= 1
    return []
```

### Two Pointers: Same Direction (Fast-Slow)

```python
def has_cycle(head):
    """Floyd's tortoise and hare."""
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False

def remove_duplicates(arr):
    """In-place dedup of sorted array."""
    if not arr:
        return 0
    slow = 0
    for fast in range(1, len(arr)):
        if arr[fast] != arr[slow]:
            slow += 1
            arr[slow] = arr[fast]
    return slow + 1
```

### Trapping Rain Water 🟡

```
Given elevation map, compute trapped water.

    |
  | |   |
| | | | ||
----------
[0,1,0,2,1,0,1,3,2,1,2,1]  → Answer: 6
```

```python
# Two-pointer approach: O(n) time, O(1) space
def trap(height):
    left, right = 0, len(height) - 1
    left_max = right_max = 0
    water = 0
    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                water += right_max - height[right]
            right -= 1
    return water

# Stack approach: O(n) time, O(n) space
def trap_stack(height):
    stack = []
    water = 0
    for i, h in enumerate(height):
        while stack and height[stack[-1]] < h:
            bottom = height[stack.pop()]
            if not stack:
                break
            width = i - stack[-1] - 1
            bounded = min(h, height[stack[-1]]) - bottom
            water += width * bounded
        stack.append(i)
    return water
```

---

## 2.14 Matrix Patterns 🟢

```python
def spiral_order(matrix):
    """Spiral traversal of m×n matrix."""
    result = []
    if not matrix:
        return result
    top, bottom, left, right = 0, len(matrix) - 1, 0, len(matrix[0]) - 1
    while top <= bottom and left <= right:
        for c in range(left, right + 1):
            result.append(matrix[top][c])
        top += 1
        for r in range(top, bottom + 1):
            result.append(matrix[r][right])
        right -= 1
        if top <= bottom:
            for c in range(right, left - 1, -1):
                result.append(matrix[bottom][c])
            bottom -= 1
        if left <= right:
            for r in range(bottom, top - 1, -1):
                result.append(matrix[r][left])
            left += 1
    return result

def rotate_90(matrix):
    """Rotate n×n matrix 90° clockwise in-place."""
    n = len(matrix)
    # Transpose
    for i in range(n):
        for j in range(i + 1, n):
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
    # Reverse each row
    for row in matrix:
        row.reverse()

def search_sorted_matrix(matrix, target):
    """Search in row-sorted & col-sorted matrix. O(m+n)."""
    if not matrix:
        return False
    r, c = 0, len(matrix[0]) - 1
    while r < len(matrix) and c >= 0:
        if matrix[r][c] == target:
            return True
        elif matrix[r][c] > target:
            c -= 1
        else:
            r += 1
    return False

def set_zeroes(matrix):
    """If element is 0, set entire row & column to 0. O(1) space."""
    m, n = len(matrix), len(matrix[0])
    first_row_zero = any(matrix[0][j] == 0 for j in range(n))
    first_col_zero = any(matrix[i][0] == 0 for i in range(m))

    for i in range(1, m):
        for j in range(1, n):
            if matrix[i][j] == 0:
                matrix[i][0] = matrix[0][j] = 0

    for i in range(1, m):
        for j in range(1, n):
            if matrix[i][0] == 0 or matrix[0][j] == 0:
                matrix[i][j] = 0

    if first_row_zero:
        for j in range(n):
            matrix[0][j] = 0
    if first_col_zero:
        for i in range(m):
            matrix[i][0] = 0
```

---

## Summary Table

| Structure | Access | Insert Front | Insert End | Delete | Space | Best For |
|-----------|--------|-------------|------------|--------|-------|----------|
| Array | O(1) | O(n) | O(1)* | O(n) | O(n) | Random access |
| Singly LL | O(n) | O(1) | O(n)† | O(1)‡ | O(n) | Frequent insertions |
| Doubly LL | O(n) | O(1) | O(1) | O(1)‡ | O(n) | Bidirectional traversal |
| Circular LL | O(n) | O(1) | O(1)† | O(1)‡ | O(n) | Round-robin |
| Stack | O(n) | O(1) | - | O(1) | O(n) | LIFO operations |
| Queue | O(n) | - | O(1) | O(1) | O(n) | FIFO operations |
| Deque | O(n) | O(1) | O(1) | O(1) | O(n) | Both-end operations |
| Circular Buffer | O(1) | O(1) | O(1) | O(1) | O(k) | Fixed-size streaming |
| Difference Array | O(n) | - | - | - | O(n) | Range updates |
| Bit Array | O(1) | - | - | - | O(n/8) | Boolean flags, membership |
| Sparse Array | O(1) | - | - | O(1) | O(nnz) | Mostly-empty data |

*\* amortized for dynamic arrays*
*† O(1) with tail pointer*
*‡ after finding the node*

---

[← Previous: Complexity Analysis](01-complexity-and-fundamentals.md) | [Next: Trees →](03-data-structures-trees.md)
