# Chapter 11: Mathematical & Bit Manipulation Algorithms 🟢🟡🔴

---

# Part A: Number Theory & Math

## 11.1 GCD and LCM 🟢

```python
# Euclidean Algorithm
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

def lcm(a, b):
    return a * b // gcd(a, b)

# Extended Euclidean: find x, y such that ax + by = gcd(a, b)
def extended_gcd(a, b):
    if a == 0:
        return b, 0, 1
    g, x1, y1 = extended_gcd(b % a, a)
    x = y1 - (b // a) * x1
    y = x1
    return g, x, y

# Modular Inverse: a⁻¹ mod m (exists when gcd(a, m) = 1)
def mod_inverse(a, m):
    g, x, _ = extended_gcd(a % m, m)
    if g != 1:
        return None  # Inverse doesn't exist
    return x % m
```

---

## 11.2 Prime Numbers 🟢

### Sieve of Eratosthenes

```
Mark composites:
2  3  [4] 5  [6] 7  [8] [9] [10] 11 [12] 13 [14] [15] [16] 17 ...
         ×        ×  ×   ×         ×         ×   ×    ×
Primes: 2, 3, 5, 7, 11, 13, 17, 19, 23, ...
```

```python
def sieve_of_eratosthenes(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False

    for i in range(2, int(n**0.5) + 1):
        if is_prime[i]:
            for j in range(i*i, n + 1, i):
                is_prime[j] = False

    return [i for i in range(n + 1) if is_prime[i]]

# Time: O(n log log n), Space: O(n)
```

### Segmented Sieve

For finding primes in range [L, R] where R can be very large.

```python
def segmented_sieve(L, R):
    limit = int(R**0.5) + 1
    small_primes = sieve_of_eratosthenes(limit)

    is_prime = [True] * (R - L + 1)
    if L == 0: is_prime[0] = False
    if L <= 1: is_prime[1 - L] = False

    for p in small_primes:
        start = max(p * p, ((L + p - 1) // p) * p)
        for j in range(start, R + 1, p):
            is_prime[j - L] = False

    return [i + L for i in range(R - L + 1) if is_prime[i]]
```

### Primality Testing

```python
# Trial division: O(√n)
def is_prime(n):
    if n < 2: return False
    if n < 4: return True
    if n % 2 == 0 or n % 3 == 0: return False
    i = 5
    while i * i <= n:
        if n % i == 0 or n % (i + 2) == 0:
            return False
        i += 6
    return True

# Miller-Rabin (probabilistic): O(k log²n)
def miller_rabin(n, k=10):
    if n < 2: return False
    if n == 2 or n == 3: return True
    if n % 2 == 0: return False

    # Write n-1 as 2^r × d
    r, d = 0, n - 1
    while d % 2 == 0:
        r += 1
        d //= 2

    import random
    for _ in range(k):
        a = random.randrange(2, n - 1)
        x = pow(a, d, n)

        if x == 1 or x == n - 1:
            continue

        for _ in range(r - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                break
        else:
            return False

    return True  # Probably prime

# Deterministic Miller-Rabin for numbers < 3.3×10²⁴
def is_prime_deterministic(n):
    if n < 2: return False
    for p in [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37]:
        if n == p: return True
        if n % p == 0: return False

    r, d = 0, n - 1
    while d % 2 == 0:
        r += 1
        d //= 2

    for a in [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37]:
        x = pow(a, d, n)
        if x == 1 or x == n - 1:
            continue
        for _ in range(r - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                break
        else:
            return False
    return True
```

### Prime Factorization

```python
def prime_factors(n):
    factors = {}
    d = 2
    while d * d <= n:
        while n % d == 0:
            factors[d] = factors.get(d, 0) + 1
            n //= d
        d += 1
    if n > 1:
        factors[n] = factors.get(n, 0) + 1
    return factors

# Pollard's Rho for large number factorization
def pollard_rho(n):
    if n % 2 == 0: return 2
    import random, math
    x = random.randint(2, n - 1)
    y = x
    c = random.randint(1, n - 1)
    d = 1
    while d == 1:
        x = (x * x + c) % n
        y = (y * y + c) % n
        y = (y * y + c) % n
        d = math.gcd(abs(x - y), n)
    return d if d != n else None
```

---

## 11.3 Modular Arithmetic 🟡

```python
# Properties:
# (a + b) mod m = ((a mod m) + (b mod m)) mod m
# (a × b) mod m = ((a mod m) × (b mod m)) mod m
# (a - b) mod m = ((a mod m) - (b mod m) + m) mod m
# (a / b) mod m = (a × b⁻¹) mod m  (where b⁻¹ is modular inverse)

# Fast Modular Exponentiation: a^b mod m
def mod_pow(a, b, m):
    result = 1
    a %= m
    while b > 0:
        if b & 1:
            result = result * a % m
        a = a * a % m
        b >>= 1
    return result

# This is built into Python: pow(a, b, m)
```

---

## 11.4 Euler's Totient Function 🟡

φ(n) = count of numbers from 1 to n that are coprime to n.

```python
def euler_totient(n):
    result = n
    p = 2
    while p * p <= n:
        if n % p == 0:
            while n % p == 0:
                n //= p
            result -= result // p
        p += 1
    if n > 1:
        result -= result // n
    return result

# Sieve version for all values 1..n
def euler_totient_sieve(n):
    phi = list(range(n + 1))
    for i in range(2, n + 1):
        if phi[i] == i:  # i is prime
            for j in range(i, n + 1, i):
                phi[j] -= phi[j] // i
    return phi
```

---

## 11.5 Chinese Remainder Theorem 🟡

Solve system: x ≡ a₁ (mod m₁), x ≡ a₂ (mod m₂), ...

```python
def chinese_remainder(remainders, moduli):
    """Find x such that x ≡ r[i] (mod m[i]) for all i"""
    M = 1
    for m in moduli:
        M *= m

    x = 0
    for r, m in zip(remainders, moduli):
        Mi = M // m
        yi = mod_inverse(Mi, m)
        x += r * Mi * yi

    return x % M

# Example: x ≡ 2 (mod 3), x ≡ 3 (mod 5), x ≡ 2 (mod 7)
print(chinese_remainder([2, 3, 2], [3, 5, 7]))  # 23
```

---

## 11.6 Combinatorics 🟡

```python
# nCr mod p (Lucas' theorem for prime p)
def nCr_mod(n, r, p):
    if r > n: return 0
    if r == 0 or r == n: return 1

    # Precompute factorials and inverses
    fact = [1] * (n + 1)
    for i in range(1, n + 1):
        fact[i] = fact[i-1] * i % p

    inv_fact = [1] * (n + 1)
    inv_fact[n] = pow(fact[n], p - 2, p)
    for i in range(n - 1, -1, -1):
        inv_fact[i] = inv_fact[i+1] * (i+1) % p

    return fact[n] * inv_fact[r] % p * inv_fact[n-r] % p

# Pascal's Triangle
def pascal_triangle(n):
    C = [[0] * (n + 1) for _ in range(n + 1)]
    for i in range(n + 1):
        C[i][0] = 1
        for j in range(1, i + 1):
            C[i][j] = C[i-1][j-1] + C[i-1][j]
    return C

# Catalan Numbers: C(n) = (2n)! / ((n+1)! × n!)
# Counts: balanced parentheses, BSTs with n nodes, paths in grid, etc.
def catalan(n):
    if n <= 1: return 1
    dp = [0] * (n + 1)
    dp[0] = dp[1] = 1
    for i in range(2, n + 1):
        for j in range(i):
            dp[i] += dp[j] * dp[i-1-j]
    return dp[n]
```

---

## 11.7 Matrix Exponentiation 🟡

Compute matrix^n in O(k³ log n) for k×k matrix. Used for linear recurrences.

```python
import numpy as np

def matrix_mult(A, B, mod=10**9 + 7):
    """Multiply two matrices mod m"""
    rows_A, cols_A = len(A), len(A[0])
    cols_B = len(B[0])
    C = [[0] * cols_B for _ in range(rows_A)]
    for i in range(rows_A):
        for j in range(cols_B):
            for k in range(cols_A):
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % mod
    return C

def matrix_pow(M, n, mod=10**9 + 7):
    """Compute M^n mod m"""
    size = len(M)
    result = [[1 if i == j else 0 for j in range(size)] for i in range(size)]

    while n > 0:
        if n & 1:
            result = matrix_mult(result, M, mod)
        M = matrix_mult(M, M, mod)
        n >>= 1

    return result

# Fibonacci in O(log n)
def fib_matrix(n, mod=10**9 + 7):
    if n <= 1: return n
    M = [[1, 1], [1, 0]]
    result = matrix_pow(M, n - 1, mod)
    return result[0][0]

# General: any linear recurrence f(n) = c₁f(n-1) + c₂f(n-2) + ... + cₖf(n-k)
# Transformation matrix:
# | c₁ c₂ ... cₖ |   | f(n-1) |   | f(n)   |
# | 1  0  ... 0  | × | f(n-2) | = | f(n-1) |
# | 0  1  ... 0  |   | ...    |   | f(n-2) |
# | ...          |   | f(n-k) |   | ...    |
```

---

## 11.8 Fast Fourier Transform (FFT) 🔴

Multiply two polynomials in O(n log n) instead of O(n²).

```python
import cmath

def fft(a, invert=False):
    n = len(a)
    if n == 1:
        return a

    a_even = fft(a[::2], invert)
    a_odd = fft(a[1::2], invert)

    angle = 2 * cmath.pi / n * (-1 if invert else 1)
    w = 1
    wn = cmath.exp(1j * angle)

    result = [0] * n
    for i in range(n // 2):
        result[i] = a_even[i] + w * a_odd[i]
        result[i + n // 2] = a_even[i] - w * a_odd[i]
        if invert:
            result[i] /= 2
            result[i + n // 2] /= 2
        w *= wn

    return result

def polynomial_multiply(a, b):
    """Multiply polynomials a and b"""
    n = 1
    while n < len(a) + len(b):
        n <<= 1

    fa = a + [0] * (n - len(a))
    fb = b + [0] * (n - len(b))

    fa = fft(fa)
    fb = fft(fb)

    fc = [fa[i] * fb[i] for i in range(n)]
    c = fft(fc, invert=True)

    return [round(x.real) for x in c]

# Example: (1 + 2x)(3 + 4x) = 3 + 10x + 8x²
print(polynomial_multiply([1, 2], [3, 4]))  # [3, 10, 8]
```

### Number Theoretic Transform (NTT)

Like FFT but works in modular arithmetic — no floating point errors.

```python
def ntt(a, mod, g, invert=False):
    """NTT using primitive root g of mod"""
    n = len(a)
    j = 0
    for i in range(1, n):
        bit = n >> 1
        while j & bit:
            j ^= bit
            bit >>= 1
        j ^= bit
        if i < j:
            a[i], a[j] = a[j], a[i]

    length = 2
    while length <= n:
        w = pow(g, (mod - 1) // length, mod)
        if invert:
            w = pow(w, mod - 2, mod)

        for i in range(0, n, length):
            wn = 1
            for k in range(length // 2):
                u = a[i + k]
                v = a[i + k + length // 2] * wn % mod
                a[i + k] = (u + v) % mod
                a[i + k + length // 2] = (u - v) % mod
                wn = wn * w % mod
        length <<= 1

    if invert:
        inv_n = pow(n, mod - 2, mod)
        a = [x * inv_n % mod for x in a]

    return a
```

---

## 11.9 Linear Algebra Algorithms 🟡

```python
# Gaussian Elimination
def gauss(matrix):
    """Solve system of linear equations"""
    n = len(matrix)
    for col in range(n):
        # Find pivot
        max_row = max(range(col, n), key=lambda r: abs(matrix[r][col]))
        matrix[col], matrix[max_row] = matrix[max_row], matrix[col]

        if abs(matrix[col][col]) < 1e-9:
            continue  # No pivot in this column

        # Eliminate below
        for row in range(col + 1, n):
            factor = matrix[row][col] / matrix[col][col]
            for j in range(col, n + 1):
                matrix[row][j] -= factor * matrix[col][j]

    # Back substitution
    solution = [0] * n
    for i in range(n - 1, -1, -1):
        solution[i] = matrix[i][n]
        for j in range(i + 1, n):
            solution[i] -= matrix[i][j] * solution[j]
        solution[i] /= matrix[i][i]

    return solution
```

---

# Part B: Bit Manipulation

## 11.10 Bit Basics 🟢

```
Binary representation of 13: 1101

Operations:
  AND (&):   1101 & 1010 = 1000 (bits both 1)
  OR  (|):   1101 | 1010 = 1111 (either bit 1)
  XOR (^):   1101 ^ 1010 = 0111 (bits differ)
  NOT (~):   ~1101 = ...0010 (flip all bits)
  Left Shift (<<):   1101 << 2 = 110100 (multiply by 4)
  Right Shift (>>):  1101 >> 1 = 0110   (divide by 2)
```

### Essential Bit Tricks

```python
# Check if i-th bit is set
def is_bit_set(n, i):
    return bool(n & (1 << i))

# Set i-th bit
def set_bit(n, i):
    return n | (1 << i)

# Clear i-th bit
def clear_bit(n, i):
    return n & ~(1 << i)

# Toggle i-th bit
def toggle_bit(n, i):
    return n ^ (1 << i)

# Check if power of 2
def is_power_of_two(n):
    return n > 0 and (n & (n - 1)) == 0

# Count set bits (Brian Kernighan's)
def count_bits(n):
    count = 0
    while n:
        n &= n - 1  # Remove lowest set bit
        count += 1
    return count

# Python built-in: bin(n).count('1')

# Lowest set bit
def lowest_set_bit(n):
    return n & (-n)

# Clear lowest set bit
def clear_lowest_set_bit(n):
    return n & (n - 1)

# Check if n has exactly one bit set
def exactly_one_bit(n):
    return n > 0 and (n & (n - 1)) == 0

# Get all submasks of a mask
def all_submasks(mask):
    submask = mask
    while submask > 0:
        yield submask
        submask = (submask - 1) & mask
    yield 0
```

---

## 11.11 XOR Tricks 🟢

```python
# Find the single number (all others appear twice)
def single_number(nums):
    result = 0
    for n in nums:
        result ^= n
    return result
# Works because a ^ a = 0 and a ^ 0 = a

# Find two numbers that appear once (all others appear twice)
def two_singles(nums):
    xor = 0
    for n in nums:
        xor ^= n

    # xor = a ^ b. Find a bit where a and b differ
    diff_bit = xor & (-xor)  # Lowest set bit

    a = b = 0
    for n in nums:
        if n & diff_bit:
            a ^= n
        else:
            b ^= n

    return a, b

# Swap without temp variable
def swap(a, b):
    a ^= b
    b ^= a
    a ^= b
    return a, b

# XOR from 1 to n (pattern repeats every 4)
def xor_to_n(n):
    """XOR of all numbers from 0 to n"""
    mod = n % 4
    if mod == 0: return n
    if mod == 1: return 1
    if mod == 2: return n + 1
    return 0
```

---

## 11.12 Gray Code 🟡

Adjacent codes differ by only one bit.

```
n=3 Gray Code:
  Binary  Gray
  000     000
  001     001
  010     011
  011     010
  100     110
  101     111
  110     101
  111     100
```

```python
def gray_code(n):
    return [i ^ (i >> 1) for i in range(1 << n)]

def gray_to_binary(gray):
    binary = gray
    mask = gray >> 1
    while mask:
        binary ^= mask
        mask >>= 1
    return binary
```

---

## 11.13 Bit Manipulation in Algorithms 🟡

```python
# Subset enumeration with bitmask
def enumerate_subsets(n):
    """Generate all subsets of {0, 1, ..., n-1}"""
    for mask in range(1 << n):
        subset = [i for i in range(n) if mask & (1 << i)]
        yield subset

# Fast multiplication/division by powers of 2
# n * 8 = n << 3
# n / 4 = n >> 2

# Absolute value without branching
def abs_val(n):
    mask = n >> 31  # All 1s if negative, all 0s if positive (32-bit)
    return (n + mask) ^ mask

# Min/Max without branching (for 32-bit integers)
def bit_min(a, b):
    diff = a - b
    return b + (diff & (diff >> 31))

# Modulo power of 2
# n % 8 = n & 7  (works for powers of 2)

# Check if numbers have opposite signs
def opposite_signs(a, b):
    return (a ^ b) < 0

# Next power of 2
def next_power_of_2(n):
    n -= 1
    n |= n >> 1
    n |= n >> 2
    n |= n >> 4
    n |= n >> 8
    n |= n >> 16
    return n + 1

# Reverse bits
def reverse_bits(n, bits=32):
    result = 0
    for _ in range(bits):
        result = (result << 1) | (n & 1)
        n >>= 1
    return result
```

---

## 11.14 Bitwise Sieve 🟡

More memory-efficient sieve using bits instead of bytes.

```python
def bitwise_sieve(n):
    size = (n >> 5) + 1  # 32 bits per int
    sieve = [0] * size

    def is_composite(i):
        return sieve[i >> 5] & (1 << (i & 31))

    def set_composite(i):
        sieve[i >> 5] |= (1 << (i & 31))

    primes = []
    for i in range(2, n + 1):
        if not is_composite(i):
            primes.append(i)
            for j in range(i * i, n + 1, i):
                set_composite(j)

    return primes
```

---

## 11.15 Algorithm Summary

| Algorithm | Time | Space | Use Case |
|-----------|------|-------|----------|
| Sieve of Eratosthenes | O(n log log n) | O(n) | All primes up to n |
| Miller-Rabin | O(k log²n) | O(1) | Primality test |
| Extended GCD | O(log min(a,b)) | O(1) | Modular inverse |
| Matrix Exponentiation | O(k³ log n) | O(k²) | Linear recurrences |
| FFT | O(n log n) | O(n) | Polynomial multiplication |
| Gaussian Elimination | O(n³) | O(n²) | Systems of equations |
| Brian Kernighan's | O(set bits) | O(1) | Counting bits |
| Baby-step Giant-step | O(√n) | O(√n) | Discrete logarithm |
| Möbius Inversion | O(n log n) | O(n) | Multiplicative functions |
| Burnside's Lemma | Varies | Varies | Counting under symmetry |
| Lagrange Interpolation | O(n²) | O(n) | Polynomial from points |
| Simplex Method | O(2^n) worst | O(mn) | Linear programming |

---

## Additional Mathematical Algorithms

### Discrete Logarithm (Baby-step Giant-step) 🟡

Find x such that g^x ≡ h (mod p). O(√p) time and space.

```python
import math

def baby_step_giant_step(g, h, p):
    """Find x such that g^x ≡ h (mod p), or -1 if none exists"""
    m = math.isqrt(p) + 1

    # Baby step: compute g^j for j = 0..m-1
    table = {}
    power = 1
    for j in range(m):
        table[power] = j
        power = power * g % p

    # Giant step: compute h * (g^{-m})^i for i = 0..m-1
    factor = pow(g, -m, p)  # g^{-m} mod p (modular inverse)
    gamma = h
    for i in range(m):
        if gamma in table:
            return i * m + table[gamma]
        gamma = gamma * factor % p

    return -1

# Example: Find x such that 3^x ≡ 7 (mod 41)
print(baby_step_giant_step(3, 7, 41))
```

### Möbius Function & Möbius Inversion 🟡

```python
def mobius_sieve(n):
    """Compute Möbius function μ(i) for i = 0..n
    μ(1) = 1
    μ(n) = 0 if n has squared prime factor
    μ(n) = (-1)^k if n is product of k distinct primes
    """
    mu = [0] * (n + 1)
    mu[1] = 1
    is_prime = [True] * (n + 1)
    primes = []

    for i in range(2, n + 1):
        if is_prime[i]:
            primes.append(i)
            mu[i] = -1  # i is prime → one prime factor
        for p in primes:
            if i * p > n:
                break
            is_prime[i * p] = False
            if i % p == 0:
                mu[i * p] = 0  # p² divides i*p
                break
            mu[i * p] = -mu[i]

    return mu

def euler_totient_via_mobius(n, mu):
    """φ(n) = Σ_{d|n} μ(d) * (n/d) — Möbius inversion"""
    result = 0
    for d in range(1, n + 1):
        if n % d == 0:
            result += mu[d] * (n // d)
    return result

# Count coprime pairs using Möbius
def count_coprime_pairs(n, mu):
    """Count pairs (a,b) with 1 ≤ a,b ≤ n and gcd(a,b) = 1"""
    total = 0
    for d in range(1, n + 1):
        total += mu[d] * (n // d) ** 2
    return total
```

### Inclusion-Exclusion Principle 🟡

```python
def inclusion_exclusion(n, sets_sizes, intersections):
    """
    |A₁ ∪ A₂ ∪ ... ∪ Aₖ| = Σ|Aᵢ| - Σ|Aᵢ∩Aⱼ| + Σ|Aᵢ∩Aⱼ∩Aₖ| - ...
    """
    from itertools import combinations
    k = len(sets_sizes)
    total = 0
    for size in range(1, k + 1):
        for combo in combinations(range(k), size):
            inter_size = intersections[combo]
            if size % 2 == 1:
                total += inter_size
            else:
                total -= inter_size
    return total

def count_derangements(n):
    """Count permutations with no fixed points using inclusion-exclusion"""
    # D(n) = n! * Σ_{k=0}^{n} (-1)^k / k!
    from math import factorial
    result = 0
    for k in range(n + 1):
        result += (-1) ** k * factorial(n) // factorial(k)
    return result

def euler_totient_ie(n):
    """Euler's totient via inclusion-exclusion on prime factors"""
    result = n
    p = 2
    temp = n
    while p * p <= temp:
        if temp % p == 0:
            while temp % p == 0:
                temp //= p
            result -= result // p
        p += 1
    if temp > 1:
        result -= result // temp
    return result

print(count_derangements(5))  # 44
print(euler_totient_ie(12))   # 4 (numbers coprime to 12: 1,5,7,11)
```

### Burnside's Lemma & Pólya Enumeration 🟡

Count distinct objects under group actions (symmetry).

```python
from math import gcd

def burnside_necklaces(n, k):
    """
    Count distinct necklaces of n beads with k colors.
    Necklace = equivalence class under rotation.

    By Burnside: |orbits| = (1/|G|) Σ_{g∈G} |Fix(g)|
    For cyclic group: Σ_{d|n} φ(n/d) * k^d / n
    """
    def euler_totient(m):
        result = m
        p = 2
        temp = m
        while p * p <= temp:
            if temp % p == 0:
                while temp % p == 0:
                    temp //= p
                result -= result // p
            p += 1
        if temp > 1:
            result -= result // temp
        return result

    total = 0
    for d in range(1, n + 1):
        if n % d == 0:
            total += euler_totient(n // d) * pow(k, d)
    return total // n

def burnside_bracelets(n, k):
    """Bracelets = necklaces considering flips (dihedral group)"""
    necklaces = burnside_necklaces(n, k) * n  # Undo division by n
    if n % 2 == 0:
        flip_fix = (n // 2) * pow(k, n // 2) + (n // 2) * pow(k, n // 2 + 1)
    else:
        flip_fix = n * pow(k, (n + 1) // 2)
    return (necklaces + flip_fix) // (2 * n)

print(burnside_necklaces(4, 2))  # 4: WWWW, WWWB, WWBB, WBWB, WBBB, BBBB → 6?
# Actually: 4 beads, 2 colors → 6 distinct necklaces
```

### Generating Functions (Conceptual) 🟡

```
Ordinary Generating Function (OGF):
  A(x) = Σ aₙ xⁿ  where aₙ is count of structures of size n

Examples:
  1/(1-x) = 1 + x + x² + x³ + ...     (sequence of all 1s)
  1/(1-x)² = 1 + 2x + 3x² + 4x³ + ... (natural numbers)
  eˣ = 1 + x + x²/2! + x³/3! + ...    (EGF for labeled structures)

Coin Change as Generating Function:
  Coins {1, 5, 10, 25}
  GF = 1/((1-x)(1-x⁵)(1-x¹⁰)(1-x²⁵))
  Coefficient of xⁿ = number of ways to make change for n cents

Fibonacci: F(x) = x/(1-x-x²)

Catalan: C(x) = (1 - √(1-4x))/(2x)

Partition function: P(x) = Π_{k=1}^∞ 1/(1-x^k)

Practical usage: multiply polynomials (FFT) to count combinations.
```

### Lagrange Interpolation 🟡

Reconstruct polynomial from n+1 points in O(n²).

```python
def lagrange_interpolation(points, x, mod=None):
    """
    Given points [(x₀,y₀), (x₁,y₁), ...], evaluate polynomial at x.
    If mod is given, works in modular arithmetic.
    """
    n = len(points)
    result = 0

    for i in range(n):
        xi, yi = points[i]
        term = yi
        for j in range(n):
            if i != j:
                xj = points[j][0]
                if mod:
                    term = term * ((x - xj) * pow(xi - xj, -1, mod)) % mod
                else:
                    term = term * (x - xj) / (xi - xj)
        result = (result + term) % mod if mod else result + term

    return result

# Example: find polynomial passing through (1,1), (2,4), (3,9) → it's x²
points = [(1, 1), (2, 4), (3, 9)]
print(lagrange_interpolation(points, 5))  # 25.0

# Useful for: computing f(n) when f is polynomial of degree d
# and you know f(0), f(1), ..., f(d)
```

### Linear Programming (Simplex Method) 🔴

```python
import numpy as np

def simplex(c, A, b):
    """
    Maximize c^T x subject to Ax ≤ b, x ≥ 0

    c: objective coefficients (1D array)
    A: constraint matrix (2D array)
    b: RHS of constraints (1D array)

    Returns: optimal value, optimal x
    """
    m, n = A.shape

    # Add slack variables: [A | I] x' = b
    tableau = np.zeros((m + 1, n + m + 1))
    tableau[:m, :n] = A
    tableau[:m, n:n+m] = np.eye(m)
    tableau[:m, -1] = b
    tableau[-1, :n] = -c  # Objective row

    basis = list(range(n, n + m))  # Slack variables as initial basis

    while True:
        # Find entering variable (most negative in objective row)
        pivot_col = np.argmin(tableau[-1, :-1])
        if tableau[-1, pivot_col] >= -1e-10:
            break  # Optimal

        # Find leaving variable (minimum ratio test)
        ratios = []
        for i in range(m):
            if tableau[i, pivot_col] > 1e-10:
                ratios.append((tableau[i, -1] / tableau[i, pivot_col], i))

        if not ratios:
            return float('inf'), None  # Unbounded

        _, pivot_row = min(ratios)
        basis[pivot_row] = pivot_col

        # Pivot
        tableau[pivot_row] /= tableau[pivot_row, pivot_col]
        for i in range(m + 1):
            if i != pivot_row:
                tableau[i] -= tableau[i, pivot_col] * tableau[pivot_row]

    # Extract solution
    x = np.zeros(n)
    for i, var in enumerate(basis):
        if var < n:
            x[var] = tableau[i, -1]

    return tableau[-1, -1], x

# Example: Maximize 3x + 5y subject to x ≤ 4, 2y ≤ 12, 3x + 5y ≤ 25
c = np.array([3.0, 5.0])
A = np.array([[1, 0], [0, 2], [3, 5]], dtype=float)
b = np.array([4.0, 12.0, 25.0])
opt_val, opt_x = simplex(c, A, b)
print(f"Optimal: {opt_val:.1f} at x={opt_x}")
```

### Stirling Numbers 🟡

```python
def stirling_second(n, k):
    """
    S(n, k) = number of ways to partition n elements into k non-empty subsets.
    S(n,k) = k * S(n-1,k) + S(n-1,k-1)
    """
    dp = [[0] * (k + 1) for _ in range(n + 1)]
    dp[0][0] = 1
    for i in range(1, n + 1):
        for j in range(1, min(i, k) + 1):
            dp[i][j] = j * dp[i-1][j] + dp[i-1][j-1]
    return dp[n][k]

def bell_number(n):
    """B(n) = total # of partitions = Σ S(n,k) for k=0..n"""
    return sum(stirling_second(n, k) for k in range(n + 1))

def stirling_first(n, k):
    """
    s(n, k) = (unsigned) number of permutations of n with k cycles.
    |s(n,k)| = (n-1)*|s(n-1,k)| + |s(n-1,k-1)|
    """
    dp = [[0] * (k + 1) for _ in range(n + 1)]
    dp[0][0] = 1
    for i in range(1, n + 1):
        for j in range(1, min(i, k) + 1):
            dp[i][j] = (i - 1) * dp[i-1][j] + dp[i-1][j-1]
    return dp[n][k]

print(stirling_second(4, 2))  # 7: ways to split {1,2,3,4} into 2 non-empty parts
print(bell_number(4))          # 15
```

### Newton's Method & Numerical Methods 🟡

```python
def newtons_method(f, df, x0, tol=1e-12, max_iter=100):
    """Find root of f(x) = 0 using Newton's method"""
    x = x0
    for _ in range(max_iter):
        fx = f(x)
        dfx = df(x)
        if abs(dfx) < 1e-15:
            break
        x_new = x - fx / dfx
        if abs(x_new - x) < tol:
            return x_new
        x = x_new
    return x

def simpsons_rule(f, a, b, n=1000):
    """Approximate ∫ₐᵇ f(x)dx using Simpson's rule"""
    if n % 2:
        n += 1
    h = (b - a) / n
    result = f(a) + f(b)
    for i in range(1, n):
        x = a + i * h
        result += (4 if i % 2 else 2) * f(x)
    return result * h / 3

# Example: √2 via Newton's method
import math
root = newtons_method(lambda x: x**2 - 2, lambda x: 2*x, 1.0)
print(f"√2 ≈ {root}")  # 1.4142135623730951

# Example: ∫₀¹ x² dx = 1/3
print(simpsons_rule(lambda x: x**2, 0, 1))  # ≈ 0.3333...
```

### Continued Fractions & Stern-Brocot Tree 🟡

```python
def to_continued_fraction(p, q, max_terms=50):
    """Convert rational p/q to continued fraction [a0; a1, a2, ...]"""
    cf = []
    for _ in range(max_terms):
        if q == 0:
            break
        a = p // q
        cf.append(a)
        p, q = q, p - a * q
    return cf

def from_continued_fraction(cf):
    """Convert continued fraction back to rational p/q"""
    if not cf:
        return 0, 1
    p, q = cf[-1], 1
    for a in reversed(cf[:-1]):
        p, q = a * p + q, p
    return p, q

def best_rational_approximation(x, max_denom):
    """Find best rational approximation p/q to real x with q ≤ max_denom"""
    # Using Stern-Brocot tree / mediants
    a, b = 0, 1  # Left bound: a/b = 0/1
    c, d = 1, 0  # Right bound: c/d = 1/0 = ∞
    best_p, best_q = 0, 1

    while True:
        # Mediant
        p, q = a + c, b + d
        if q > max_denom:
            break
        if abs(p/q - x) < abs(best_p/best_q - x):
            best_p, best_q = p, q
        if p/q < x:
            a, b = p, q
        elif p/q > x:
            c, d = p, q
        else:
            break

    return best_p, best_q

# π ≈ 355/113
print(to_continued_fraction(355, 113))  # [3, 7, 16] → 3 + 1/(7 + 1/16)
print(best_rational_approximation(3.14159265, 1000))  # (355, 113)
```

---

[← Previous: Greedy, Backtracking & D&C](10-greedy-backtracking-divide-conquer.md) | [Next: Network Flow & Geometry →](12-network-flow-and-geometry.md)
