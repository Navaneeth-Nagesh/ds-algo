# Chapter 6: Sorting & Searching Algorithms 🟢🟡

---

# Part A: Sorting Algorithms

## Why Sorting Matters

Sorting is the most studied problem in computer science. Sorted data enables:
- Binary search (O(log n) instead of O(n))
- Efficient merging and deduplication
- Finding closest pairs, medians, percentiles
- Database indexing and query optimization

---

## 6.1 Bubble Sort 🟢

Repeatedly swap adjacent elements if they're in the wrong order. The largest element "bubbles" to the end each pass.

```
Pass 1: [5, 3, 8, 1, 2] → [3, 5, 1, 2, 8]   (8 bubbled to end)
Pass 2: [3, 5, 1, 2, 8] → [3, 1, 2, 5, 8]   (5 bubbled)
Pass 3: [3, 1, 2, 5, 8] → [1, 2, 3, 5, 8]   (3 bubbled)
Pass 4: [1, 2, 3, 5, 8] → [1, 2, 3, 5, 8]   (already sorted, stop)
```

```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        swapped = False
        for j in range(n - 1 - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = True
        if not swapped:  # Optimization: stop if no swaps
            break
    return arr
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n) | O(n²) | O(n²) | O(1) | Yes |

---

## 6.2 Selection Sort 🟢

Find the minimum element and place it at the beginning. Repeat for the remaining array.

```
[29, 10, 14, 37, 13]
 → Find min (10), swap with pos 0: [10, 29, 14, 37, 13]
 → Find min (13), swap with pos 1: [10, 13, 14, 37, 29]
 → Find min (14), swap with pos 2: [10, 13, 14, 37, 29]
 → Find min (29), swap with pos 3: [10, 13, 14, 29, 37]
```

```python
def selection_sort(arr):
    n = len(arr)
    for i in range(n):
        min_idx = i
        for j in range(i + 1, n):
            if arr[j] < arr[min_idx]:
                min_idx = j
        arr[i], arr[min_idx] = arr[min_idx], arr[i]
    return arr
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n²) | O(n²) | O(n²) | O(1) | No |

Always does exactly n(n-1)/2 comparisons regardless of input.

---

## 6.3 Insertion Sort 🟢

Build the sorted array one element at a time by inserting each element into its correct position.

```
[5, 3, 8, 1, 2]
 → Insert 3: [3, 5, 8, 1, 2]    (shifted 5 right)
 → Insert 8: [3, 5, 8, 1, 2]    (already in place)
 → Insert 1: [1, 3, 5, 8, 2]    (shifted 3, 5, 8 right)
 → Insert 2: [1, 2, 3, 5, 8]    (shifted 3, 5, 8 right)
```

```python
def insertion_sort(arr):
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key
    return arr
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n) | O(n²) | O(n²) | O(1) | Yes |

**Best for:** Small arrays (< 10-50 elements), nearly sorted data, online sorting (data arrives one at a time).

---

## 6.4 Merge Sort 🟢

Divide array in half, sort each half recursively, then merge the sorted halves.

```
[38, 27, 43, 3, 9, 82, 10]
       /                \
[38, 27, 43, 3]    [9, 82, 10]
    /       \          /      \
[38, 27]  [43, 3]  [9, 82]  [10]
 /    \    /    \    /    \     |
[38] [27] [43] [3] [9]  [82] [10]
 \    /    \    /    \    /     |
[27, 38]  [3, 43]  [9, 82]  [10]
    \       /          \      /
[3, 27, 38, 43]    [9, 10, 82]
       \                /
[3, 9, 10, 27, 38, 43, 82]
```

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
        if left[i] <= right[j]:  # <= for stability
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result

# In-place merge sort (to reduce space)
def merge_sort_inplace(arr, left, right):
    if right - left <= 1:
        return
    mid = (left + right) // 2
    merge_sort_inplace(arr, left, mid)
    merge_sort_inplace(arr, mid, right)
    merge_inplace(arr, left, mid, right)
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |

**Key advantage:** Guaranteed O(n log n) regardless of input. Excellent for linked lists (no random access needed). Good for external sorting (large files).

---

## 6.5 Quick Sort 🟢

Pick a "pivot", partition array into elements < pivot and > pivot, recursively sort both sides.

```
Pivot = 4:
[3, 6, 8, 10, 1, 2, 4]
         ↓ partition
[3, 1, 2] [4] [6, 8, 10]
  ↓ sort        ↓ sort
[1, 2, 3] [4] [6, 8, 10]
```

```python
def quick_sort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[len(arr) // 2]  # Middle element as pivot
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return quick_sort(left) + middle + quick_sort(right)

# In-place Quick Sort (Lomuto partition)
def quick_sort_inplace(arr, low, high):
    if low < high:
        pi = partition(arr, low, high)
        quick_sort_inplace(arr, low, pi - 1)
        quick_sort_inplace(arr, pi + 1, high)

def partition(arr, low, high):
    pivot = arr[high]
    i = low - 1
    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1

# Hoare partition (fewer swaps)
def hoare_partition(arr, low, high):
    pivot = arr[(low + high) // 2]
    i, j = low - 1, high + 1
    while True:
        i += 1
        while arr[i] < pivot:
            i += 1
        j -= 1
        while arr[j] > pivot:
            j -= 1
        if i >= j:
            return j
        arr[i], arr[j] = arr[j], arr[i]

# 3-Way Quick Sort (handles duplicates well)
def quick_sort_3way(arr, low, high):
    if low >= high:
        return
    lt, gt = low, high
    pivot = arr[low]
    i = low + 1
    while i <= gt:
        if arr[i] < pivot:
            arr[lt], arr[i] = arr[i], arr[lt]
            lt += 1
            i += 1
        elif arr[i] > pivot:
            arr[gt], arr[i] = arr[i], arr[gt]
            gt -= 1
        else:
            i += 1
    quick_sort_3way(arr, low, lt - 1)
    quick_sort_3way(arr, gt + 1, high)
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n log n) | O(n log n) | O(n²) | O(log n) | No |

**Why it's popular:** Fastest in practice due to cache locality and small constants. The O(n²) worst case is avoided with random pivot selection.

---

## 6.6 Heap Sort 🟡

Build a max-heap from the array, then repeatedly extract the maximum.

```python
def heap_sort(arr):
    n = len(arr)

    # Build max-heap (bottom-up)
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)

    # Extract elements one by one
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]  # Move max to end
        heapify(arr, i, 0)

    return arr

def heapify(arr, n, i):
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    if left < n and arr[left] > arr[largest]:
        largest = left
    if right < n and arr[right] > arr[largest]:
        largest = right

    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n log n) | O(n log n) | O(n log n) | O(1) | No |

**Advantage:** O(1) space, guaranteed O(n log n). **Disadvantage:** Poor cache performance, not stable.

---

## 6.7 Counting Sort 🟢

Count occurrences of each value. Only works for integers in a known range.

```
Input:  [4, 2, 2, 8, 3, 3, 1]
Count:  index: 0  1  2  3  4  5  6  7  8
        count: 0  1  2  2  1  0  0  0  1
Output: [1, 2, 2, 3, 3, 4, 8]
```

```python
def counting_sort(arr):
    if not arr:
        return arr
    max_val = max(arr)
    count = [0] * (max_val + 1)

    for num in arr:
        count[num] += 1

    result = []
    for i, c in enumerate(count):
        result.extend([i] * c)

    return result

# Stable version (preserves order of equal elements)
def counting_sort_stable(arr, max_val):
    count = [0] * (max_val + 1)
    for num in arr:
        count[num] += 1

    # Prefix sum
    for i in range(1, len(count)):
        count[i] += count[i - 1]

    output = [0] * len(arr)
    for num in reversed(arr):  # Reversed for stability
        count[num] -= 1
        output[count[num]] = num

    return output
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |

Where k = range of values. **Limitation:** Only for non-negative integers or must map values.

---

## 6.8 Radix Sort 🟡

Sort by each digit, starting from the least significant. Uses counting sort as a subroutine.

```
Input: [170, 45, 75, 90, 802, 24, 2, 66]

Sort by ones digit: [170, 90, 802, 2, 24, 45, 75, 66]
Sort by tens digit:  [802, 2, 24, 45, 66, 170, 75, 90]
Sort by hundreds:    [2, 24, 45, 66, 75, 90, 170, 802]
```

```python
def radix_sort(arr):
    if not arr:
        return arr
    max_val = max(arr)
    exp = 1
    while max_val // exp > 0:
        counting_sort_by_digit(arr, exp)
        exp *= 10
    return arr

def counting_sort_by_digit(arr, exp):
    n = len(arr)
    output = [0] * n
    count = [0] * 10

    for num in arr:
        digit = (num // exp) % 10
        count[digit] += 1

    for i in range(1, 10):
        count[i] += count[i - 1]

    for i in range(n - 1, -1, -1):
        digit = (arr[i] // exp) % 10
        count[digit] -= 1
        output[count[digit]] = arr[i]

    for i in range(n):
        arr[i] = output[i]
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(nk) | O(nk) | O(nk) | O(n+k) | Yes |

Where k = number of digits. Beats O(n log n) when k is small!

---

## 6.9 Bucket Sort 🟡

Distribute elements into buckets, sort each bucket, concatenate.

```
Input: [0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12]

Bucket 0 (0.0-0.2): [0.17, 0.12]
Bucket 1 (0.2-0.4): [0.39, 0.26, 0.21]
Bucket 2 (0.4-0.6): []
Bucket 3 (0.6-0.8): [0.78, 0.72]
Bucket 4 (0.8-1.0): [0.94]

Sort each bucket, concatenate:
[0.12, 0.17, 0.21, 0.26, 0.39, 0.72, 0.78, 0.94]
```

```python
def bucket_sort(arr, num_buckets=10):
    if not arr:
        return arr

    min_val, max_val = min(arr), max(arr)
    bucket_range = (max_val - min_val) / num_buckets + 1

    buckets = [[] for _ in range(num_buckets)]
    for num in arr:
        idx = int((num - min_val) / bucket_range)
        buckets[idx].append(num)

    result = []
    for bucket in buckets:
        result.extend(sorted(bucket))  # Sort each bucket

    return result
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n+k) | O(n+k) | O(n²) | O(n+k) | Yes |

**Best for:** Uniformly distributed data over a known range.

---

## 6.10 Tim Sort 🟡

Python's built-in sort! Hybrid of merge sort and insertion sort.

**Key ideas:**
1. Find natural "runs" (already sorted subsequences) in the data
2. Extend short runs using insertion sort (minimum run length ~32-64)
3. Merge runs using a sophisticated merge strategy
4. Uses "galloping mode" to speed up merges when one run dominates

```python
# Python's built-in sort IS Tim Sort
arr = [5, 3, 8, 1, 2, 7, 4, 6]
arr.sort()           # In-place Tim Sort
sorted_arr = sorted(arr)  # Returns new sorted list
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n) | O(n log n) | O(n log n) | O(n) | Yes |

**Why it's great:** Adapts to existing order in data. Nearly sorted data → nearly O(n).

---

## 6.11 Shell Sort 🟡

Generalization of insertion sort using decreasing gap sequences.

```python
def shell_sort(arr):
    n = len(arr)
    gap = n // 2

    while gap > 0:
        for i in range(gap, n):
            temp = arr[i]
            j = i
            while j >= gap and arr[j - gap] > temp:
                arr[j] = arr[j - gap]
                j -= gap
            arr[j] = temp
        gap //= 2

    return arr
```

| Best | Average | Worst | Space | Stable |
|------|---------|-------|-------|--------|
| O(n log n) | Depends on gap sequence | O(n²) | O(1) | No |

Gap sequence matters: Knuth's (1, 4, 13, 40, ...) gives O(n^1.5).

---

## 6.12 Comb Sort 🟡

Like bubble sort but uses decreasing gaps (similar to shell sort's relationship to insertion sort).

```python
def comb_sort(arr):
    n = len(arr)
    gap = n
    shrink = 1.3
    sorted_flag = False

    while not sorted_flag:
        gap = max(1, int(gap / shrink))
        sorted_flag = (gap == 1)

        for i in range(n - gap):
            if arr[i] > arr[i + gap]:
                arr[i], arr[i + gap] = arr[i + gap], arr[i]
                sorted_flag = False

    return arr
```

---

## 6.13 Cocktail Shaker Sort (Bidirectional Bubble Sort) 🟢

Bubble sort that alternates direction — forward pass then backward pass.

```python
def cocktail_sort(arr):
    n = len(arr)
    start, end = 0, n - 1
    swapped = True

    while swapped:
        swapped = False
        for i in range(start, end):
            if arr[i] > arr[i + 1]:
                arr[i], arr[i + 1] = arr[i + 1], arr[i]
                swapped = True
        end -= 1

        if not swapped:
            break

        swapped = False
        for i in range(end, start, -1):
            if arr[i] < arr[i - 1]:
                arr[i], arr[i - 1] = arr[i - 1], arr[i]
                swapped = True
        start += 1

    return arr
```

---

## 6.14 Gnome Sort 🟢

Like insertion sort but moves backward by swaps instead of shifts. "A garden gnome sorting flower pots."

```python
def gnome_sort(arr):
    i = 0
    while i < len(arr):
        if i == 0 or arr[i] >= arr[i - 1]:
            i += 1
        else:
            arr[i], arr[i - 1] = arr[i - 1], arr[i]
            i -= 1
    return arr
```

---

## 6.15 Cycle Sort 🟡

Minimizes the number of writes. Optimal for situations where writing is expensive.

```python
def cycle_sort(arr):
    writes = 0
    for start in range(len(arr) - 1):
        item = arr[start]
        pos = start
        for i in range(start + 1, len(arr)):
            if arr[i] < item:
                pos += 1

        if pos == start:
            continue

        while item == arr[pos]:
            pos += 1
        arr[pos], item = item, arr[pos]
        writes += 1

        while pos != start:
            pos = start
            for i in range(start + 1, len(arr)):
                if arr[i] < item:
                    pos += 1
            while item == arr[pos]:
                pos += 1
            arr[pos], item = item, arr[pos]
            writes += 1

    return arr
```

---

## 6.16 Pigeonhole Sort 🟡

Like counting sort, for integer keys in a small range. Creates a "hole" for each possible value.

```python
def pigeonhole_sort(arr):
    min_val, max_val = min(arr), max(arr)
    size = max_val - min_val + 1
    holes = [0] * size

    for x in arr:
        holes[x - min_val] += 1

    i = 0
    for j in range(size):
        while holes[j] > 0:
            arr[i] = j + min_val
            holes[j] -= 1
            i += 1
    return arr
```

---

## 6.17 Bitonic Sort 🟡

A parallel sorting algorithm that works by creating **bitonic sequences** (first increasing then decreasing) and merging them.

```
Bitonic sequence: [1, 3, 5, 7, 6, 4, 2, 0]
                   ↑ increasing ↑ ↓ decreasing ↓
```

```python
def bitonic_sort(arr, low, count, ascending):
    if count > 1:
        k = count // 2
        bitonic_sort(arr, low, k, True)       # Sort ascending
        bitonic_sort(arr, low + k, k, False)  # Sort descending
        bitonic_merge(arr, low, count, ascending)

def bitonic_merge(arr, low, count, ascending):
    if count > 1:
        k = count // 2
        for i in range(low, low + k):
            if (arr[i] > arr[i + k]) == ascending:
                arr[i], arr[i + k] = arr[i + k], arr[i]
        bitonic_merge(arr, low, k, ascending)
        bitonic_merge(arr, low + k, k, ascending)
```

**Time:** O(n log²n) — not optimal sequentially, but highly parallelizable → O(log²n) with n processors.

---

## 6.18 Pancake Sort 🟡

Sort by flipping prefixes of the array (like flipping a stack of pancakes with a spatula).

```python
def pancake_sort(arr):
    n = len(arr)
    for size in range(n, 1, -1):
        # Find max in arr[0..size-1]
        max_idx = arr.index(max(arr[:size]))

        if max_idx != size - 1:
            # Flip to bring max to front
            arr[:max_idx + 1] = arr[:max_idx + 1][::-1]
            # Flip to move max to correct position
            arr[:size] = arr[:size][::-1]

    return arr
```

---

## 6.19 Patience Sort 🟡

Based on the card game "Patience" (Solitaire). Related to the Longest Increasing Subsequence.

```python
import heapq
from bisect import bisect_left

def patience_sort(arr):
    piles = []
    for num in arr:
        pos = bisect_left([pile[-1] for pile in piles], num)
        if pos == len(piles):
            piles.append([num])
        else:
            piles[pos].append(num)

    # Merge piles using a min-heap
    result = []
    heap = [(pile.pop(0), i) for i, pile in enumerate(piles)]
    heapq.heapify(heap)

    while heap:
        val, pile_idx = heapq.heappop(heap)
        result.append(val)
        if piles[pile_idx]:
            heapq.heappush(heap, (piles[pile_idx].pop(0), pile_idx))

    return result
```

**Bonus:** The number of piles equals the length of the Longest Increasing Subsequence!

---

## 6.20 Intro Sort (Introspective Sort) 🟡

Used by C++ STL `std::sort`. Starts with quick sort, switches to heap sort if recursion depth exceeds O(log n), and uses insertion sort for small subarrays.

```python
import math

def intro_sort(arr):
    max_depth = 2 * int(math.log2(len(arr) + 1))
    _intro_sort(arr, 0, len(arr) - 1, max_depth)
    return arr

def _intro_sort(arr, low, high, depth_limit):
    size = high - low + 1
    if size <= 16:
        insertion_sort_range(arr, low, high)
    elif depth_limit == 0:
        heap_sort_range(arr, low, high)
    else:
        pivot = partition(arr, low, high)
        _intro_sort(arr, low, pivot - 1, depth_limit - 1)
        _intro_sort(arr, pivot + 1, high, depth_limit - 1)
```

---

## 6.21 Tree Sort 🟡

Insert all elements into a BST, then do inorder traversal. O(n log n) average, O(n²) worst (if BST degenerates). With self-balancing tree, guaranteed O(n log n).

---

## 6.22 Odd-Even Sort (Brick Sort) 🟡

Parallel version of bubble sort. Alternates between odd-indexed and even-indexed comparisons.

```python
def odd_even_sort(arr):
    n = len(arr)
    sorted_flag = False
    while not sorted_flag:
        sorted_flag = True
        for i in range(1, n - 1, 2):  # Odd indices
            if arr[i] > arr[i + 1]:
                arr[i], arr[i + 1] = arr[i + 1], arr[i]
                sorted_flag = False
        for i in range(0, n - 1, 2):  # Even indices
            if arr[i] > arr[i + 1]:
                arr[i], arr[i + 1] = arr[i + 1], arr[i]
                sorted_flag = False
    return arr
```

---

## 6.23 Strand Sort 🟡

Extracts sorted subsequences ("strands") from the input and merges them.

```python
def strand_sort(arr):
    if not arr:
        return []

    result = []
    while arr:
        # Extract a strand
        strand = [arr.pop(0)]
        i = 0
        while i < len(arr):
            if arr[i] >= strand[-1]:
                strand.append(arr.pop(i))
            else:
                i += 1
        # Merge strand with result
        result = merge(result, strand)

    return result
```

---

## 6.24 Smooth Sort 🔴

Heap sort variant using Leonardo heaps. Achieves O(n) on already sorted data while maintaining O(n log n) worst case and O(1) space.

---

## 6.25 Block Sort (WikiSort) 🔴

An in-place stable O(n log n) merge sort using O(1) extra space. Works by using internal buffering.

---

## 6.26 Library Sort 🟡

Insert elements with gaps between them (like shelving library books), allowing efficient insertion sort.

---

## 6.27 Flash Sort 🟡

A distribution sort that estimates where each element should go using the distribution of values. O(n) for uniformly distributed data.

---

## 6.28 Bogo Sort (Permutation Sort) 🟢

Keep shuffling randomly until sorted. **Expected time: O(n × n!)**. Never use this!

```python
import random

def bogo_sort(arr):
    while arr != sorted(arr):
        random.shuffle(arr)
    return arr
```

---

## 6.29 Stooge Sort 🟢

Recursively sorts the first 2/3, then last 2/3, then first 2/3 again. Terrible complexity: O(n^(log3/log1.5)) ≈ O(n^2.71).

---

## 6.30 Sleep Sort 🟢

Create a thread for each element that sleeps for (value) seconds, then prints. Elements print in order!

```python
import threading, time

def sleep_sort(arr):
    result = []
    def sleep_and_add(x):
        time.sleep(x * 0.001)  # Scale down for speed
        result.append(x)

    threads = [threading.Thread(target=sleep_and_add, args=(x,)) for x in arr]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return result
```

Not practical, but a fun concurrency exercise!

---

## 6.31 External Sort 🟡

For data too large to fit in memory:
1. Read chunks that fit in memory
2. Sort each chunk (create sorted "runs")
3. Merge runs using k-way merge with a min-heap

```python
import heapq

def k_way_merge(sorted_lists):
    """Merge k sorted lists using a min-heap"""
    heap = []
    for i, lst in enumerate(sorted_lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))

    result = []
    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)
        if elem_idx + 1 < len(sorted_lists[list_idx]):
            next_val = sorted_lists[list_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))

    return result
```

---

## 6.32 Quick Select (Selection Algorithm) 🟢

Find the k-th smallest element in O(n) average time. Like quicksort, but only recurse into one partition.

```python
def quick_select(arr, k):
    """Find k-th smallest element (0-indexed)"""
    if len(arr) == 1:
        return arr[0]

    pivot = arr[len(arr) // 2]
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]

    if k < len(left):
        return quick_select(left, k)
    elif k < len(left) + len(middle):
        return pivot
    else:
        return quick_select(right, k - len(left) - len(middle))

# Median of Medians — O(n) WORST CASE selection
def median_of_medians(arr, k):
    if len(arr) <= 5:
        return sorted(arr)[k]

    # Divide into groups of 5, find median of each
    medians = []
    for i in range(0, len(arr), 5):
        group = sorted(arr[i:i+5])
        medians.append(group[len(group) // 2])

    # Find median of medians recursively
    pivot = median_of_medians(medians, len(medians) // 2)

    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]

    if k < len(left):
        return median_of_medians(left, k)
    elif k < len(left) + len(middle):
        return pivot
    else:
        return median_of_medians(right, k - len(left) - len(middle))
```

---

# Part B: Searching Algorithms

## 6.33 Linear Search 🟢

Check every element one by one. Works on any data.

```python
def linear_search(arr, target):
    for i, val in enumerate(arr):
        if val == target:
            return i
    return -1
```

**Time:** O(n) | **Space:** O(1)

---

## 6.34 Binary Search 🟢

For **sorted arrays** — repeatedly halve the search space.

```
Target = 23
[2, 5, 8, 12, 16, 23, 38, 56, 72, 91]
 L                 M                 R    → 23 > 16, go right
                      L    M        R     → 23 < 56, go left
                      L M  R              → 23 < 38, go left
                      LMR                 → Found 23!
```

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = left + (right - left) // 2  # Avoid overflow
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1

# Find leftmost (first) occurrence
def bisect_left(arr, target):
    left, right = 0, len(arr)
    while left < right:
        mid = (left + right) // 2
        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid
    return left

# Find rightmost (last) occurrence
def bisect_right(arr, target):
    left, right = 0, len(arr)
    while left < right:
        mid = (left + right) // 2
        if arr[mid] <= target:
            left = mid + 1
        else:
            right = mid
    return left

# Binary search on answer (very common pattern!)
def find_minimum_capacity(weights, days):
    """Find minimum ship capacity to ship all packages within 'days' days"""
    def can_ship(capacity):
        current_weight = 0
        needed_days = 1
        for w in weights:
            if current_weight + w > capacity:
                needed_days += 1
                current_weight = 0
            current_weight += w
        return needed_days <= days

    left, right = max(weights), sum(weights)
    while left < right:
        mid = (left + right) // 2
        if can_ship(mid):
            right = mid
        else:
            left = mid + 1
    return left
```

**Time:** O(log n) | **Space:** O(1) iterative, O(log n) recursive

---

## 6.35 Ternary Search 🟡

For finding the maximum/minimum of a **unimodal function** (one peak or valley). Divides into thirds.

```python
def ternary_search(f, left, right, precision=1e-9):
    """Find maximum of unimodal function f on [left, right]"""
    while right - left > precision:
        m1 = left + (right - left) / 3
        m2 = right - (right - left) / 3
        if f(m1) < f(m2):
            left = m1
        else:
            right = m2
    return (left + right) / 2
```

**Time:** O(log₃ n) ≈ O(log n)

---

## 6.36 Jump Search 🟡

Jump ahead by √n steps, then linear search backward. For sorted arrays.

```python
import math

def jump_search(arr, target):
    n = len(arr)
    step = int(math.sqrt(n))
    prev = 0

    # Jump ahead
    while arr[min(step, n) - 1] < target:
        prev = step
        step += int(math.sqrt(n))
        if prev >= n:
            return -1

    # Linear search in block
    while arr[prev] < target:
        prev += 1
        if prev == min(step, n):
            return -1

    return prev if arr[prev] == target else -1
```

**Time:** O(√n) | Best for: when binary search's random access is expensive.

---

## 6.37 Interpolation Search 🟡

Like binary search, but estimates position based on value distribution. For uniformly distributed sorted data.

```python
def interpolation_search(arr, target):
    low, high = 0, len(arr) - 1

    while low <= high and arr[low] <= target <= arr[high]:
        if arr[high] == arr[low]:
            if arr[low] == target:
                return low
            return -1

        # Estimate position
        pos = low + int(((target - arr[low]) * (high - low)) / (arr[high] - arr[low]))

        if arr[pos] == target:
            return pos
        elif arr[pos] < target:
            low = pos + 1
        else:
            high = pos - 1

    return -1
```

**Time:** O(log log n) for uniform data, O(n) worst case.

---

## 6.38 Exponential Search 🟡

Find the range where target exists (doubling range), then binary search within it.

```python
def exponential_search(arr, target):
    if arr[0] == target:
        return 0

    n = len(arr)
    bound = 1
    while bound < n and arr[bound] <= target:
        bound *= 2

    # Binary search in [bound/2, min(bound, n-1)]
    left = bound // 2
    right = min(bound, n - 1)
    return binary_search_range(arr, target, left, right)
```

**Time:** O(log n) | **Best for:** Unbounded/infinite lists, unknown size arrays.

---

## 6.39 Fibonacci Search 🟡

Uses Fibonacci numbers to divide the array (instead of halving). Useful when multiplication is costly.

```python
def fibonacci_search(arr, target):
    n = len(arr)
    fib2 = 0  # F(k-2)
    fib1 = 1  # F(k-1)
    fib = fib1 + fib2  # F(k)

    while fib < n:
        fib2 = fib1
        fib1 = fib
        fib = fib1 + fib2

    offset = -1
    while fib > 1:
        i = min(offset + fib2, n - 1)
        if arr[i] < target:
            fib = fib1
            fib1 = fib2
            fib2 = fib - fib1
            offset = i
        elif arr[i] > target:
            fib = fib2
            fib1 = fib1 - fib2
            fib2 = fib - fib1
        else:
            return i

    if fib1 and offset + 1 < n and arr[offset + 1] == target:
        return offset + 1

    return -1
```

---

## 6.40 Two Pointer Technique 🟢

Use two pointers moving from both ends (or different speeds) to solve array problems.

```python
# Find pair with target sum in sorted array
def pair_sum(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        s = arr[left] + arr[right]
        if s == target:
            return (left, right)
        elif s < target:
            left += 1
        else:
            right -= 1
    return None

# Remove duplicates from sorted array (in-place)
def remove_duplicates(arr):
    if not arr:
        return 0
    write = 1
    for read in range(1, len(arr)):
        if arr[read] != arr[read - 1]:
            arr[write] = arr[read]
            write += 1
    return write

# Three Sum — find all triplets summing to zero
def three_sum(nums):
    nums.sort()
    result = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        left, right = i + 1, len(nums) - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    return result
```

---

## Additional Sorting Algorithms

### Tournament Sort 🟡

Build a tournament tree (winner tree), extract winner, replace, replay.

```python
def tournament_sort(arr):
    """O(n log n) — optimal replacement for selection sort"""
    import math
    n = len(arr)
    size = 1
    while size < n:
        size *= 2
    tree = [float('inf')] * (2 * size)
    for i in range(n):
        tree[size + i] = arr[i]
    for i in range(size - 1, 0, -1):
        tree[i] = min(tree[2*i], tree[2*i+1])

    result = []
    for _ in range(n):
        result.append(tree[1])
        # Find and remove the winner
        idx = tree.index(tree[1], size)
        tree[idx] = float('inf')
        idx //= 2
        while idx >= 1:
            tree[idx] = min(tree[2*idx], tree[2*idx+1])
            idx //= 2
    return result
```

### Sample Sort 🟡

Generalization of quicksort: pick s pivots, partition into s+1 buckets. **Best parallel sorting algorithm in practice.**

```python
import random

def sample_sort(arr, num_buckets=4):
    if len(arr) <= num_buckets:
        return sorted(arr)

    # Sample and sort pivots
    samples = sorted(random.sample(arr, min(len(arr)-1, num_buckets * 3)))
    pivots = [samples[i * len(samples) // num_buckets] for i in range(1, num_buckets)]

    # Partition into buckets
    from bisect import bisect_left
    buckets = [[] for _ in range(num_buckets)]
    for x in arr:
        buckets[bisect_left(pivots, x)].append(x)

    # Recursively sort each bucket (in parallel in real implementation)
    return sum([sample_sort(b, num_buckets) for b in buckets], [])
```

### American Flag Sort (MSD Radix, In-Place) 🟡

```python
def american_flag_sort(arr, byte_idx=0, max_bytes=4):
    """In-place MSD radix sort — O(n × max_bytes)"""
    if len(arr) <= 1 or byte_idx >= max_bytes:
        return
    BUCKETS = 256
    counts = [0] * BUCKETS
    for x in arr:
        digit = (x >> (8 * (max_bytes - 1 - byte_idx))) & 0xFF
        counts[digit] += 1
    offsets = [0] * BUCKETS
    for i in range(1, BUCKETS):
        offsets[i] = offsets[i-1] + counts[i-1]
    # Use offsets to place elements
    # (simplified — full implementation swaps in-place)
```

---

## Additional Search Algorithms

### IDA* (Iterative Deepening A*) 🟡

A* with iterative deepening on f-value. Uses O(depth) memory vs A*'s exponential.

```python
def ida_star(start, goal, heuristic, successors):
    def search(path, g, threshold):
        node = path[-1]
        f = g + heuristic(node, goal)
        if f > threshold:
            return f, None
        if node == goal:
            return -1, path[:]

        min_threshold = float('inf')
        for neighbor, cost in successors(node):
            if neighbor not in path:
                path.append(neighbor)
                t, result = search(path, g + cost, threshold)
                if result is not None:
                    return -1, result
                min_threshold = min(min_threshold, t)
                path.pop()
        return min_threshold, None

    threshold = heuristic(start, goal)
    path = [start]
    while True:
        t, result = search(path, 0, threshold)
        if result is not None:
            return result
        if t == float('inf'):
            return None
        threshold = t
```

### Branch and Bound 🟡

Systematic search with pruning via bounds for optimization problems.

```python
def branch_and_bound_knapsack(weights, values, capacity):
    n = len(weights)
    best = [0]

    def bound(idx, weight, value):
        """Fractional knapsack upper bound"""
        if weight > capacity:
            return 0
        b = value
        w = weight
        for i in range(idx, n):
            if w + weights[i] <= capacity:
                w += weights[i]
                b += values[i]
            else:
                b += values[i] * (capacity - w) / weights[i]
                break
        return b

    def solve(idx, weight, value):
        if weight > capacity:
            return
        best[0] = max(best[0], value)
        if idx >= n:
            return
        if bound(idx, weight, value) <= best[0]:
            return  # Prune!
        solve(idx + 1, weight + weights[idx], value + values[idx])  # Include
        solve(idx + 1, weight, value)  # Exclude

    # Sort by value/weight ratio for better bounds
    items = sorted(zip(weights, values), key=lambda x: x[1]/x[0], reverse=True)
    weights, values = zip(*items)
    solve(0, 0, 0)
    return best[0]
```

### Beam Search 🟡

BFS that keeps only the top-k nodes at each level. Used in NLP and speech recognition.

```python
def beam_search(start, goal, successors, heuristic, beam_width=3):
    beam = [(heuristic(start, goal), [start])]
    while beam:
        next_beam = []
        for _, path in beam:
            node = path[-1]
            if node == goal:
                return path
            for neighbor, cost in successors(node):
                new_path = path + [neighbor]
                score = heuristic(neighbor, goal)
                next_beam.append((score, new_path))
        # Keep only top beam_width candidates
        next_beam.sort()
        beam = next_beam[:beam_width]
    return None
```

### IDDFS (Iterative Deepening DFS) 🟢

Combines DFS space efficiency O(d) with BFS completeness.

```python
def iddfs(start, goal, successors, max_depth=100):
    for depth_limit in range(max_depth):
        result = dls(start, goal, successors, depth_limit)
        if result is not None:
            return result
    return None

def dls(node, goal, successors, limit):
    if node == goal:
        return [node]
    if limit <= 0:
        return None
    for neighbor in successors(node):
        result = dls(neighbor, goal, successors, limit - 1)
        if result is not None:
            return [node] + result
    return None
```

---

## Dutch National Flag (3-Way Partition) 🟢

```
Sort array with 3 distinct values in O(n) time, O(1) space.
Used in: Sort Colors (LeetCode 75), QuickSort with duplicates.

  [2,0,2,1,1,0]  →  [0,0,1,1,2,2]
```

```python
def dutch_national_flag(arr, pivot=1):
    """3-way partition: elements < pivot, == pivot, > pivot."""
    lo, mid, hi = 0, 0, len(arr) - 1
    while mid <= hi:
        if arr[mid] < pivot:
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo += 1
            mid += 1
        elif arr[mid] > pivot:
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi -= 1
        else:
            mid += 1
    return arr

# Sort Colors: sort [0,1,2] array
def sort_colors(nums):
    dutch_national_flag(nums, pivot=1)
```

---

## Fisher-Yates (Knuth) Shuffle 🟢

```python
import random

def fisher_yates_shuffle(arr):
    """Uniform random shuffle in O(n) time, O(1) space.
    Each permutation is equally likely."""
    for i in range(len(arr) - 1, 0, -1):
        j = random.randint(0, i)
        arr[i], arr[j] = arr[j], arr[i]
    return arr

# Reservoir Sampling: pick k items from unknown-length stream
def reservoir_sample(stream, k):
    """O(n) time, O(k) space. Each item has k/n probability."""
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

## Binary Search Patterns 🟢

```python
# Pattern 1: Search in Rotated Sorted Array
def search_rotated(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid
        if nums[lo] <= nums[mid]:  # left half sorted
            if nums[lo] <= target < nums[mid]:
                hi = mid - 1
            else:
                lo = mid + 1
        else:  # right half sorted
            if nums[mid] < target <= nums[hi]:
                lo = mid + 1
            else:
                hi = mid - 1
    return -1

# Pattern 2: Find First/Last Position
def find_first(nums, target):
    lo, hi, result = 0, len(nums) - 1, -1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            result = mid
            hi = mid - 1  # keep searching left
        elif nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return result

def find_last(nums, target):
    lo, hi, result = 0, len(nums) - 1, -1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            result = mid
            lo = mid + 1  # keep searching right
        elif nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return result

# Pattern 3: Find Peak Element
def find_peak(nums):
    lo, hi = 0, len(nums) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if nums[mid] < nums[mid + 1]:
            lo = mid + 1
        else:
            hi = mid
    return lo

# Pattern 4: Binary Search on Answer Space
def min_eating_speed(piles, h):
    """Koko eating bananas: minimum speed to finish in h hours."""
    import math
    def can_finish(speed):
        return sum(math.ceil(p / speed) for p in piles) <= h

    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_finish(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo

# Pattern 5: Find Minimum in Rotated Sorted Array
def find_min_rotated(nums):
    lo, hi = 0, len(nums) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if nums[mid] > nums[hi]:
            lo = mid + 1
        else:
            hi = mid
    return nums[lo]
```

---

## Sorting Algorithm Decision Guide

```
Start
│
├── Data fits in memory?
│   ├── YES → Is n small (< 50)?
│   │         ├── YES → Insertion Sort ★
│   │         └── NO → Is stability required?
│   │                   ├── YES → Is O(n) space OK?
│   │                   │         ├── YES → Merge Sort or Tim Sort ★
│   │                   │         └── NO → Block Sort (complex!)
│   │                   └── NO → Is worst-case guarantee needed?
│   │                             ├── YES → Heap Sort
│   │                             └── NO → Quick Sort ★ (fastest in practice)
│   └── NO → External Sort (k-way merge)
│
├── Integer keys in small range?
│   └── YES → Counting Sort or Radix Sort ★
│
├── Uniformly distributed?
│   └── YES → Bucket Sort
│
├── Need parallelism?
│   └── YES → Sample Sort, Bitonic Sort, Odd-Even Merge Sort
│
├── Strings?
│   └── YES → American Flag Sort, Burst Sort, Multi-key Quicksort
│
├── Memory-constrained search?
│   └── YES → IDA*, IDDFS, Beam Search
│
└── Just use your language's built-in sort! (Almost always the right choice)

★ = Most commonly used
```

---

[← Previous: Advanced Data Structures](05-data-structures-advanced.md) | [Next: Graph Algorithms →](07-graph-algorithms.md)
