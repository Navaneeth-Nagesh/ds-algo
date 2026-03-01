# Chapter 9: String Algorithms 🟢🟡🔴

## Overview

String algorithms are fundamental in text processing, bioinformatics, search engines, and compilers.

```
Common Problems:
  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │ Pattern Matching  │    │ String Properties │    │ String Transform │
  ├──────────────────┤    ├──────────────────┤    ├──────────────────┤
  │ KMP              │    │ Palindromes      │    │ Edit Distance    │
  │ Rabin-Karp       │    │ Anagrams         │    │ String Rotation  │
  │ Boyer-Moore      │    │ Longest Repeated │    │ Run-Length Enc.  │
  │ Z-Algorithm      │    │ All Substrings   │    │ Huffman Coding   │
  │ Aho-Corasick     │    │ Suffix Array     │    │ Burrows-Wheeler  │
  └──────────────────┘    └──────────────────┘    └──────────────────┘
```

---

## 9.1 Brute Force Matching 🟢

```python
def brute_search(text, pattern):
    n, m = len(text), len(pattern)
    indices = []
    for i in range(n - m + 1):
        if text[i:i+m] == pattern:
            indices.append(i)
    return indices

# Time: O(n × m), Space: O(1)
```

---

## 9.2 KMP (Knuth-Morris-Pratt) 🟡

Never re-examine characters in the text. Uses a prefix function (failure function) to skip.

```
Pattern: "ABABAC"
Prefix Table (LPS):
  A  B  A  B  A  C
  0  0  1  2  3  0

When mismatch at position j, jump pattern to lps[j-1]
instead of starting over.
```

```python
def kmp_build_lps(pattern):
    """Build Longest Proper Prefix which is also Suffix (LPS) array"""
    m = len(pattern)
    lps = [0] * m
    length = 0
    i = 1
    while i < m:
        if pattern[i] == pattern[length]:
            length += 1
            lps[i] = length
            i += 1
        elif length > 0:
            length = lps[length - 1]
        else:
            lps[i] = 0
            i += 1
    return lps

def kmp_search(text, pattern):
    n, m = len(text), len(pattern)
    lps = kmp_build_lps(pattern)

    indices = []
    i = j = 0  # i for text, j for pattern
    while i < n:
        if text[i] == pattern[j]:
            i += 1
            j += 1

        if j == m:
            indices.append(i - j)
            j = lps[j - 1]
        elif i < n and text[i] != pattern[j]:
            if j > 0:
                j = lps[j - 1]
            else:
                i += 1

    return indices

# Example
text = "ABABDABACDABABCABAB"
pattern = "ABABCABAB"
print(kmp_search(text, pattern))  # [9]
```

**Time:** O(n + m) | **Space:** O(m)

---

## 9.3 Rabin-Karp (Rolling Hash) 🟡

Use hashing to quickly filter candidate positions, then verify matches.

```python
def rabin_karp(text, pattern, base=256, mod=10**9 + 7):
    n, m = len(text), len(pattern)
    if m > n: return []

    # Compute hash of pattern and first window of text
    pat_hash = 0
    txt_hash = 0
    h = pow(base, m - 1, mod)

    for i in range(m):
        pat_hash = (pat_hash * base + ord(pattern[i])) % mod
        txt_hash = (txt_hash * base + ord(text[i])) % mod

    indices = []
    for i in range(n - m + 1):
        if pat_hash == txt_hash:
            if text[i:i+m] == pattern:  # Verify to avoid false positives
                indices.append(i)

        if i < n - m:
            txt_hash = (txt_hash - ord(text[i]) * h) * base + ord(text[i + m])
            txt_hash %= mod

    return indices

# Average: O(n + m), Worst: O(nm) with many hash collisions
```

### Multiple Pattern Search with Rabin-Karp

Check for any of multiple patterns simultaneously by hashing all patterns first.

---

## 9.4 Boyer-Moore 🟡

Skips sections of text using two heuristics:
- **Bad Character Rule**: On mismatch, shift pattern to align the mismatched character
- **Good Suffix Rule**: Shift pattern to align a matching suffix

```python
def boyer_moore(text, pattern):
    n, m = len(text), len(pattern)
    if m > n: return []

    # Bad character table
    bad_char = {}
    for i in range(m):
        bad_char[pattern[i]] = i

    indices = []
    i = 0  # Shift of pattern with respect to text
    while i <= n - m:
        j = m - 1
        while j >= 0 and pattern[j] == text[i + j]:
            j -= 1

        if j < 0:
            indices.append(i)
            i += (m - bad_char.get(text[i + m], -1)) if i + m < n else 1
        else:
            shift = j - bad_char.get(text[i + j], -1)
            i += max(1, shift)

    return indices

# Best: O(n/m), Average: O(n), Worst: O(nm)
# Best algorithm for practical single-pattern search in English text
```

---

## 9.5 Z-Algorithm 🟡

Z[i] = length of the longest substring starting from i that is also a prefix of the string.

```
String: "aabxaab"
Z-array: [7, 1, 0, 0, 3, 1, 0]
              ^              ^
              |              |-- "aab" matches prefix "aab" (length 3)
              |-- "a" matches prefix "a" (length 1)
```

```python
def z_function(s):
    n = len(s)
    z = [0] * n
    z[0] = n
    l = r = 0

    for i in range(1, n):
        if i < r:
            z[i] = min(r - i, z[i - l])
        while i + z[i] < n and s[z[i]] == s[i + z[i]]:
            z[i] += 1
        if i + z[i] > r:
            l, r = i, i + z[i]

    return z

def z_search(text, pattern):
    """Use Z-algorithm for pattern matching"""
    combined = pattern + "$" + text
    z = z_function(combined)
    m = len(pattern)
    return [i - m - 1 for i in range(m + 1, len(combined)) if z[i] == m]

# Time: O(n + m), Space: O(n + m)
```

---

## 9.6 Aho-Corasick 🔴

Multi-pattern matching. Builds an automaton from all patterns, then scans text once.

```
Patterns: {"he", "she", "his", "hers"}
Text: "ahishers"

         root
        / | \
       h  s  ...
      / \  \
     e   i  h
     |      |
     r      e
     |
     s

Goto + Failure links → automaton for simultaneous search
```

```python
from collections import deque, defaultdict

class AhoCorasick:
    def __init__(self):
        self.goto = [{}]       # goto[state][char] → next state
        self.fail = [0]        # failure links
        self.output = [[]]     # output[state] → list of pattern indices
        self.state_count = 1

    def _new_state(self):
        self.goto.append({})
        self.fail.append(0)
        self.output.append([])
        idx = self.state_count
        self.state_count += 1
        return idx

    def build(self, patterns):
        # Build goto function (trie)
        for idx, pattern in enumerate(patterns):
            state = 0
            for ch in pattern:
                if ch not in self.goto[state]:
                    self.goto[state][ch] = self._new_state()
                state = self.goto[state][ch]
            self.output[state].append(idx)

        # Build failure function (BFS)
        queue = deque()
        for ch, s in self.goto[0].items():
            self.fail[s] = 0
            queue.append(s)

        while queue:
            u = queue.popleft()
            for ch, v in self.goto[u].items():
                queue.append(v)
                f = self.fail[u]
                while f and ch not in self.goto[f]:
                    f = self.fail[f]
                self.fail[v] = self.goto[f].get(ch, 0)
                if self.fail[v] == v:
                    self.fail[v] = 0
                self.output[v] = self.output[v] + self.output[self.fail[v]]

    def search(self, text, patterns):
        results = []
        state = 0
        for i, ch in enumerate(text):
            while state and ch not in self.goto[state]:
                state = self.fail[state]
            state = self.goto[state].get(ch, 0)
            for pattern_idx in self.output[state]:
                pos = i - len(patterns[pattern_idx]) + 1
                results.append((pos, patterns[pattern_idx]))
        return results

# Example
ac = AhoCorasick()
patterns = ["he", "she", "his", "hers"]
ac.build(patterns)
print(ac.search("ahishers", patterns))
# [(1, 'his'), (3, 'she'), (4, 'he'), (4, 'hers')]
```

**Build:** O(Σ|patterns|) | **Search:** O(|text| + matches)

---

## 9.7 Suffix Array 🟡

Sorted array of all suffixes of a string. Space-efficient alternative to suffix tree.

```
String: "banana"
Suffixes sorted:
  Index 5: "a"
  Index 3: "ana"
  Index 1: "anana"
  Index 0: "banana"
  Index 4: "na"
  Index 2: "nana"

Suffix Array: [5, 3, 1, 0, 4, 2]
```

```python
# O(n log²n) construction
def build_suffix_array(s):
    n = len(s)
    sa = list(range(n))
    rank = [ord(c) for c in s]

    k = 1
    while k < n:
        def compare_key(i):
            return (rank[i], rank[i + k] if i + k < n else -1)
        sa.sort(key=compare_key)

        new_rank = [0] * n
        for i in range(1, n):
            new_rank[sa[i]] = new_rank[sa[i-1]]
            if compare_key(sa[i]) != compare_key(sa[i-1]):
                new_rank[sa[i]] += 1
        rank = new_rank
        k *= 2

    return sa

# LCP Array (Longest Common Prefix between adjacent suffixes in sorted order)
def build_lcp(s, sa):
    n = len(s)
    rank = [0] * n
    for i in range(n):
        rank[sa[i]] = i

    lcp = [0] * n
    k = 0
    for i in range(n):
        if rank[i] == 0:
            k = 0
            continue
        j = sa[rank[i] - 1]
        while i + k < n and j + k < n and s[i+k] == s[j+k]:
            k += 1
        lcp[rank[i]] = k
        k = max(0, k - 1)

    return lcp

# Pattern search using suffix array + binary search: O(m log n)
```

---

## 9.8 Suffix Automaton (SAM) 🔴

Compact automaton that accepts all suffixes. Most powerful linear-space string structure.

```python
class SuffixAutomaton:
    class State:
        def __init__(self):
            self.len = 0
            self.link = -1
            self.transitions = {}
            self.cnt = 0  # Number of times this state's string occurs

    def __init__(self):
        self.states = [self.State()]
        self.states[0].len = 0
        self.states[0].link = -1
        self.last = 0

    def extend(self, c):
        cur = len(self.states)
        self.states.append(self.State())
        self.states[cur].len = self.states[self.last].len + 1
        self.states[cur].cnt = 1

        p = self.last
        while p != -1 and c not in self.states[p].transitions:
            self.states[p].transitions[c] = cur
            p = self.states[p].link

        if p == -1:
            self.states[cur].link = 0
        else:
            q = self.states[p].transitions[c]
            if self.states[p].len + 1 == self.states[q].len:
                self.states[cur].link = q
            else:
                clone = len(self.states)
                self.states.append(self.State())
                self.states[clone].len = self.states[p].len + 1
                self.states[clone].link = self.states[q].link
                self.states[clone].transitions = dict(self.states[q].transitions)

                while p != -1 and self.states[p].transitions.get(c) == q:
                    self.states[p].transitions[c] = clone
                    p = self.states[p].link

                self.states[q].link = clone
                self.states[cur].link = clone

        self.last = cur

    def build(self, s):
        for c in s:
            self.extend(c)

    def count_distinct_substrings(self):
        """Count distinct substrings"""
        total = 0
        for state in self.states[1:]:  # Skip initial state
            total += state.len - self.states[state.link].len
        return total

sam = SuffixAutomaton()
sam.build("abab")
print(sam.count_distinct_substrings())  # 7: a, b, ab, ba, aba, bab, abab
```

---

## 9.9 Manacher's Algorithm 🟡

Find the longest palindromic substring in O(n).

```python
def manacher(s):
    # Transform "abc" → "^#a#b#c#$"
    t = "^#" + "#".join(s) + "#$"
    n = len(t)
    p = [0] * n  # p[i] = radius of palindrome centered at i
    c = r = 0    # Center and right boundary

    for i in range(1, n - 1):
        mirror = 2 * c - i
        if i < r:
            p[i] = min(r - i, p[mirror])

        while t[i + p[i] + 1] == t[i - p[i] - 1]:
            p[i] += 1

        if i + p[i] > r:
            c, r = i, i + p[i]

    # Find the maximum
    max_len = max(p)
    center = p.index(max_len)
    start = (center - max_len) // 2
    return s[start:start + max_len]

print(manacher("babad"))    # "bab" or "aba"
print(manacher("cbbd"))     # "bb"
print(manacher("racecar"))  # "racecar"
```

---

## 9.10 String Hashing 🟡

Polynomial rolling hash for O(1) substring comparison after O(n) preprocessing.

```python
class StringHash:
    def __init__(self, s, base=131, mod=10**18 + 9):
        self.mod = mod
        self.base = base
        n = len(s)
        self.h = [0] * (n + 1)
        self.pw = [1] * (n + 1)

        for i in range(n):
            self.h[i+1] = (self.h[i] * base + ord(s[i])) % mod
            self.pw[i+1] = (self.pw[i] * base) % mod

    def get_hash(self, l, r):
        """Hash of s[l..r] (0-indexed, inclusive)"""
        return (self.h[r+1] - self.h[l] * self.pw[r-l+1]) % self.mod

    def compare(self, l1, r1, l2, r2):
        """Check if s[l1..r1] == s[l2..r2]"""
        return self.get_hash(l1, r1) == self.get_hash(l2, r2)

# Double hashing for reduced collision probability
class DoubleHash:
    def __init__(self, s):
        self.h1 = StringHash(s, base=131, mod=10**18 + 9)
        self.h2 = StringHash(s, base=137, mod=10**18 + 7)

    def get_hash(self, l, r):
        return (self.h1.get_hash(l, r), self.h2.get_hash(l, r))
```

---

## 9.11 Trie-Based String Algorithms 🟢

```python
class Trie:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.count = 0

    def insert(self, word):
        node = self
        for ch in word:
            if ch not in node.children:
                node.children[ch] = Trie()
            node = node.children[ch]
            node.count += 1
        node.is_end = True

    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end

    def starts_with(self, prefix):
        node = self._find(prefix)
        return node is not None

    def count_prefix(self, prefix):
        node = self._find(prefix)
        return node.count if node else 0

    def _find(self, prefix):
        node = self
        for ch in prefix:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node

# Autocomplete with Trie
def autocomplete(trie, prefix, max_results=10):
    node = trie._find(prefix)
    if not node:
        return []

    results = []

    def dfs(node, path):
        if len(results) >= max_results:
            return
        if node.is_end:
            results.append(prefix + "".join(path))
        for ch in sorted(node.children):
            path.append(ch)
            dfs(node.children[ch], path)
            path.pop()

    dfs(node, [])
    return results
```

---

## 9.12 Longest Common Substring 🟡

```python
# Using DP: O(nm)
def longest_common_substring(s1, s2):
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    max_len = 0
    end_pos = 0

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
                if dp[i][j] > max_len:
                    max_len = dp[i][j]
                    end_pos = i

    return s1[end_pos - max_len:end_pos]

# Using suffix array + LCP: O(n log n) for combined string
```

---

## 9.13 Regular Expression Matching 🟡

NFA-based regex matching.

```python
def is_match(text, pattern):
    """Simple regex with '.' and '*'"""
    m, n = len(text), len(pattern)
    dp = [[False] * (n + 1) for _ in range(m + 1)]
    dp[0][0] = True

    # Handle patterns like a*, a*b*, etc.
    for j in range(2, n + 1):
        if pattern[j-1] == '*':
            dp[0][j] = dp[0][j-2]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if pattern[j-1] == text[i-1] or pattern[j-1] == '.':
                dp[i][j] = dp[i-1][j-1]
            elif pattern[j-1] == '*':
                dp[i][j] = dp[i][j-2]  # Zero occurrences
                if pattern[j-2] == text[i-1] or pattern[j-2] == '.':
                    dp[i][j] = dp[i][j] or dp[i-1][j]  # One or more

    return dp[m][n]
```

---

## 9.14 Burrows-Wheeler Transform 🔴

Transforms text to make it more compressible. Foundation of bzip2 compression.

```python
def bwt(s):
    """Burrows-Wheeler Transform"""
    s = s + "$"  # End marker
    rotations = sorted(s[i:] + s[:i] for i in range(len(s)))
    return "".join(r[-1] for r in rotations)

def inverse_bwt(bwt_str):
    """Inverse BWT"""
    n = len(bwt_str)
    table = [""] * n
    for _ in range(n):
        table = sorted(bwt_str[i] + table[i] for i in range(n))
    for row in table:
        if row.endswith("$"):
            return row[:-1]

# Example
text = "banana"
encoded = bwt(text)       # "annb$aa"
decoded = inverse_bwt(encoded)  # "banana"
```

---

## 9.15 Useful String Techniques

### Minimum Rotation

Find the lexicographically smallest rotation of a string.

```python
def min_rotation(s):
    """Booth's algorithm — O(n)"""
    s = s + s
    n = len(s)
    f = [-1] * n
    k = 0
    for j in range(1, n):
        sj = s[j]
        i = f[j - 1 - k]
        while i != -1 and sj != s[k + i + 1]:
            if sj < s[k + i + 1]:
                k = j - i - 1
            i = f[i]
        if sj != s[k + i + 1]:
            if sj < s[k]:
                k = j
            f[j - k] = -1
        else:
            f[j - k] = i + 1
    return k  # Starting index of min rotation
```

### Counting Distinct Substrings

```python
# Using suffix array + LCP
def count_distinct_substrings(s):
    sa = build_suffix_array(s)
    lcp = build_lcp(s, sa)
    n = len(s)
    total = n * (n + 1) // 2  # Total substrings
    return total - sum(lcp)    # Subtract overlaps
```

---

## 9.16 Algorithm Comparison

| Algorithm | Pattern | Time | Space | Best For |
|-----------|---------|------|-------|----------|
| Brute Force | Single | O(nm) | O(1) | Very short patterns |
| KMP | Single | O(n+m) | O(m) | General single pattern |
| Boyer-Moore | Single | O(n/m) best | O(σ) | Large alphabets (English text) |
| Rabin-Karp | Single/Multi | O(n+m) avg | O(m) | Multiple pattern lengths |
| Z-Algorithm | Single | O(n+m) | O(n) | Alternative to KMP |
| Aho-Corasick | Multi | O(n+Σm) | O(Σm) | Dictionary matching |
| Suffix Array | Any | O(n log n) | O(n) | Many queries on same text |
| Suffix Automaton | Any | O(n) build | O(n) | Counting substrings |
| Manacher | Palindrome | O(n) | O(n) | Palindrome detection |
| Eertree | Palindrome | O(n) build | O(n) | All palindromic substrings |
| Bitap | Fuzzy | O(nm/w) | O(m) | Approximate matching |
| FM-Index | Any | O(m) query | O(n) | Compressed full-text index |

---

## Additional String Algorithms

### Palindromic Tree (Eertree) 🟡

Stores ALL distinct palindromic substrings in O(n) time and space.

```python
class EertreeNode:
    def __init__(self, length, suffix_link=None):
        self.length = length
        self.suffix_link = suffix_link
        self.edges = {}  # char → child node
        self.count = 0   # number of times this palindrome occurs

class Eertree:
    def __init__(self):
        # Two root nodes: imaginary string of length -1, and empty string
        self.odd_root = EertreeNode(-1)
        self.even_root = EertreeNode(0)
        self.odd_root.suffix_link = self.odd_root
        self.even_root.suffix_link = self.odd_root
        self.last = self.even_root
        self.s = [-1]  # sentinel
        self.nodes = [self.odd_root, self.even_root]

    def _get_link(self, node, i):
        """Find longest palindromic suffix where we can extend"""
        while self.s[i - node.length - 1] != self.s[i]:
            node = node.suffix_link
        return node

    def add(self, char):
        self.s.append(char)
        i = len(self.s) - 1

        cur = self._get_link(self.last, i)

        if char not in cur.edges:
            # Create new palindromic node
            new_node = EertreeNode(cur.length + 2)

            # Find suffix link for new node
            suffix = self._get_link(cur.suffix_link, i)
            new_node.suffix_link = suffix.edges.get(char, self.even_root)

            cur.edges[char] = new_node
            self.nodes.append(new_node)

        self.last = cur.edges[char]
        self.last.count += 1

    def count_palindromes(self):
        """Total number of palindromic substrings (with multiplicity)"""
        # Propagate counts from longest to shortest
        total = 0
        for node in reversed(self.nodes[2:]):
            node.suffix_link.count += node.count
            total += node.count
        return total

    def num_distinct_palindromes(self):
        return len(self.nodes) - 2  # Exclude two roots

# Example
eertree = Eertree()
for ch in "abacaba":
    eertree.add(ch)
print(f"Distinct palindromes: {eertree.num_distinct_palindromes()}")
# a, b, aba, aca, bacab, abacaba, c → 7
```

### Lyndon Factorization (Duval's Algorithm) 🟡

Decompose string into Lyndon words (lexicographically smallest rotation of itself). O(n) time, O(1) space.

```python
def duval(s):
    """Lyndon factorization of s — returns list of Lyndon words"""
    n = len(s)
    factorization = []
    i = 0
    while i < n:
        j = i + 1
        k = i
        while j < n and s[k] <= s[j]:
            if s[k] < s[j]:
                k = i
            else:
                k += 1
            j += 1
        length = j - k
        while i <= k:
            factorization.append(s[i:i+length])
            i += length
    return factorization

# Applications:
# - Lexicographically smallest rotation: concatenate s+s, find shortest Lyndon word
# - Booth's algorithm for minimum rotation
print(duval("abbaabba"))  # ['abb', 'a', 'abb', 'a']

def smallest_rotation(s):
    """Find lexicographically smallest rotation — Booth's algorithm O(n)"""
    s2 = s + s
    n = len(s)
    f = [-1] * (2 * n)
    k = 0
    for j in range(1, 2 * n):
        sj = s2[j]
        i = f[j - 1 - k]
        while i != -1 and sj != s2[k + i + 1]:
            if sj < s2[k + i + 1]:
                k = j - i - 1
            i = f[i]
        if sj != s2[k + i + 1]:
            if sj < s2[k]:
                k = j
            f[j - k] = -1
        else:
            f[j - k] = i + 1
    return s[k:] + s[:k]

print(smallest_rotation("cab"))  # "abc"
```

### Bitap Algorithm (Shift-Or / Shift-And) 🟡

Fast approximate string matching using bitwise operations. Supports up to k errors.

```python
def bitap_exact(text, pattern):
    """Exact matching using bitap — O(n·m/w) where w = word size"""
    m = len(pattern)
    if m == 0:
        return 0
    if m > 63:  # Exceeds machine word
        return text.find(pattern)

    # Bitmask for each character
    pattern_mask = {}
    for i, c in enumerate(pattern):
        if c not in pattern_mask:
            pattern_mask[c] = ~0
        pattern_mask[c] &= ~(1 << i)

    R = ~0  # State bitmask
    for i, c in enumerate(text):
        R |= pattern_mask.get(c, ~0)
        R <<= 1
        if (R & (1 << m)) == 0:
            return i - m + 1  # Match found at this position
    return -1

def bitap_fuzzy(text, pattern, max_errors):
    """Approximate matching with at most max_errors substitutions"""
    m = len(pattern)

    pattern_mask = {}
    for i, c in enumerate(pattern):
        if c not in pattern_mask:
            pattern_mask[c] = ~0
        pattern_mask[c] &= ~(1 << i)

    R = [(~0)] * (max_errors + 1)
    results = []

    for i, c in enumerate(text):
        old_R = R[:]
        mask = pattern_mask.get(c, ~0)
        R[0] = (old_R[0] << 1) | mask
        for d in range(1, max_errors + 1):
            R[d] = ((old_R[d] << 1) | mask) & (old_R[d-1] << 1) & (R[d-1] << 1) & (old_R[d-1])

        if (R[max_errors] & (1 << m)) == 0:
            results.append(i - m + 1)

    return results
```

### Sequence Alignment (Needleman-Wunsch & Smith-Waterman) 🟡

Fundamental bioinformatics algorithms for global and local sequence alignment.

```python
def needleman_wunsch(seq1, seq2, match=1, mismatch=-1, gap=-2):
    """Global alignment — O(nm) time and space"""
    n, m = len(seq1), len(seq2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]

    for i in range(n + 1):
        dp[i][0] = i * gap
    for j in range(m + 1):
        dp[0][j] = j * gap

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            score = match if seq1[i-1] == seq2[j-1] else mismatch
            dp[i][j] = max(
                dp[i-1][j-1] + score,  # Match/mismatch
                dp[i-1][j] + gap,      # Gap in seq2
                dp[i][j-1] + gap       # Gap in seq1
            )

    # Traceback
    align1, align2 = [], []
    i, j = n, m
    while i > 0 or j > 0:
        if i > 0 and j > 0:
            score = match if seq1[i-1] == seq2[j-1] else mismatch
            if dp[i][j] == dp[i-1][j-1] + score:
                align1.append(seq1[i-1])
                align2.append(seq2[j-1])
                i -= 1; j -= 1; continue
        if i > 0 and dp[i][j] == dp[i-1][j] + gap:
            align1.append(seq1[i-1])
            align2.append('-')
            i -= 1
        else:
            align1.append('-')
            align2.append(seq2[j-1])
            j -= 1

    return ''.join(reversed(align1)), ''.join(reversed(align2)), dp[n][m]

def smith_waterman(seq1, seq2, match=2, mismatch=-1, gap=-1):
    """Local alignment — finds best matching subsequence"""
    n, m = len(seq1), len(seq2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    max_score = 0
    max_pos = (0, 0)

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            score = match if seq1[i-1] == seq2[j-1] else mismatch
            dp[i][j] = max(
                0,                        # Can start fresh
                dp[i-1][j-1] + score,
                dp[i-1][j] + gap,
                dp[i][j-1] + gap
            )
            if dp[i][j] > max_score:
                max_score = dp[i][j]
                max_pos = (i, j)

    # Traceback from max_pos
    align1, align2 = [], []
    i, j = max_pos
    while dp[i][j] > 0:
        score = match if seq1[i-1] == seq2[j-1] else mismatch
        if dp[i][j] == dp[i-1][j-1] + score:
            align1.append(seq1[i-1])
            align2.append(seq2[j-1])
            i -= 1; j -= 1
        elif dp[i][j] == dp[i-1][j] + gap:
            align1.append(seq1[i-1])
            align2.append('-')
            i -= 1
        else:
            align1.append('-')
            align2.append(seq2[j-1])
            j -= 1

    return ''.join(reversed(align1)), ''.join(reversed(align2)), max_score

# Example
a1, a2, score = needleman_wunsch("GCATGCG", "GATTACA")
print(f"Global: {a1} / {a2} (score={score})")
a1, a2, score = smith_waterman("GCATGCG", "GATTACA")
print(f"Local: {a1} / {a2} (score={score})")
```

### FM-Index 🔴

Compressed full-text index based on BWT — used in bioinformatics (Bowtie, BWA).

```python
def bwt_transform(s):
    """Burrows-Wheeler Transform"""
    s = s + '$'
    rotations = [s[i:] + s[:i] for i in range(len(s))]
    rotations.sort()
    return ''.join(r[-1] for r in rotations)

def fm_index_simple(text):
    """Simplified FM-Index construction"""
    bwt = bwt_transform(text)
    n = len(bwt)

    # Count occurrences of each char up to position i
    occ = {}
    for c in set(bwt):
        occ[c] = [0] * (n + 1)
    for i in range(n):
        for c in occ:
            occ[c][i+1] = occ[c][i]
        occ[bwt[i]][i+1] += 1

    # First occurrence of each char in sorted BWT
    sorted_bwt = sorted(bwt)
    first = {}
    for i, c in enumerate(sorted_bwt):
        if c not in first:
            first[c] = i

    return bwt, occ, first

def fm_count(pattern, bwt, occ, first):
    """Count occurrences of pattern using FM-Index — O(m) time"""
    top = 0
    bottom = len(bwt) - 1
    for c in reversed(pattern):
        if c not in first:
            return 0
        top = first[c] + occ[c][top]
        bottom = first[c] + occ[c][bottom + 1] - 1
        if top > bottom:
            return 0
    return bottom - top + 1

# Example
text = "abracadabra"
bwt, occ, first = fm_index_simple(text)
print(f"'abra' occurs {fm_count('abra', bwt, occ, first)} times")  # 2
```

### SA-IS (Suffix Array via Induced Sorting) 🔴

Linear time O(n) suffix array construction — much faster than O(n log² n) naive.

```
Concept:
1. Classify suffixes as S-type or L-type
   - S-type: suffix[i] < suffix[i+1] lexicographically
   - L-type: suffix[i] > suffix[i+1] lexicographically
2. Find LMS (Left-Most S-type) positions
3. Recursively sort LMS suffixes
4. Induce full suffix array from sorted LMS

Key advantage: O(n) time with small constant factor.
Used in: libdivsufsort, many bioinformatics tools.
```

### String Period and Border Functions 🟡

```python
def compute_period(s):
    """Smallest period of string s — using failure function"""
    n = len(s)
    # KMP failure function
    fail = [0] * n
    j = 0
    for i in range(1, n):
        while j > 0 and s[i] != s[j]:
            j = fail[j-1]
        if s[i] == s[j]:
            j += 1
        fail[i] = j

    period = n - fail[-1]
    return period

def all_borders(s):
    """Find all borders (proper prefixes that are also suffixes)"""
    n = len(s)
    fail = [0] * n
    j = 0
    for i in range(1, n):
        while j > 0 and s[i] != s[j]:
            j = fail[j-1]
        if s[i] == s[j]:
            j += 1
        fail[i] = j

    borders = []
    length = fail[-1]
    while length > 0:
        borders.append(s[:length])
        length = fail[length - 1]
    return borders

print(compute_period("abcabcabc"))  # 3 ("abc" repeated)
print(all_borders("abacabab"))     # ["ab", "a"] — (not "abacab")
```

---

[← Previous: Dynamic Programming](08-dynamic-programming.md) | [Next: Greedy, Backtracking & D&C →](10-greedy-backtracking-divide-conquer.md)
