---
tags: [dsa, dsa/patterns]
aliases: [Bitwise operations, Bit tricks, Bitmask, XOR trick, Bit manipulation pattern]
updated: 2026-09-27
---
# Bit Manipulation

> [!tldr] TL;DR
> Integers are arrays of bits, and bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`) work on all of them at once in O(1). A handful of tricks cover most interview problems: **`x & (x - 1)` clears the lowest set bit**, **`x & -x` isolates it**, **XOR cancels pairs** (`a ^ a = 0`), and an integer **mask can represent a subset**. Language details matter: JavaScript bitwise ops work on **32-bit signed** integers (use `>>>` for unsigned, `BigInt` for more bits), while Python ints have **arbitrary precision**, so you mask with `0xFFFFFFFF` to simulate 32-bit behavior.

## The idea

A number in binary is a sum of powers of two:

```text
13 = 8 + 4 + 1 = 0b1101
bit index:   3 2 1 0
bit value:   1 1 0 1
```

**Two's complement** is how CPUs store signed integers: `-x` is `~x + 1` (flip all bits, add one). In 8 bits:

```text
 5 = 0000 0101
~5 = 1111 1010   (= -6)
-5 = 1111 1011   (= ~5 + 1)
```

That is why `~x == -x - 1` in every language, and why `x & -x` keeps only the lowest set bit.

## How it works

### Operators

| Op | Name | Rule per bit | Example (`a = 0b1100`, `b = 0b1010`) |
|---|---|---|---|
| `a & b` | AND | 1 if both are 1 | `0b1000` (8) |
| `a \| b` | OR | 1 if either is 1 | `0b1110` (14) |
| `a ^ b` | XOR | 1 if they differ | `0b0110` (6) |
| `~a` | NOT | flip every bit | `-13` (two's complement) |
| `a << k` | left shift | multiply by 2^k | `0b1100 << 1 = 0b11000` (24) |
| `a >> k` | arithmetic right shift | divide by 2^k, rounding toward negative infinity, keeps sign | `-8 >> 1 = -4` |
| `a >>> k` | logical right shift (JS only) | shift in zeros, result unsigned | `-1 >>> 0 = 4294967295` |

XOR laws you will use constantly: `a ^ 0 = a`, `a ^ a = 0`, XOR is commutative and associative, so order does not matter.

### Single-bit recipes (bit `i`)

| Goal | Expression |
|---|---|
| Test bit i | `(x >> i) & 1` |
| Set bit i | `x \| (1 << i)` |
| Clear bit i | `x & ~(1 << i)` |
| Toggle bit i | `x ^ (1 << i)` |
| Is x odd? | `x & 1` |
| Clear lowest set bit | `x & (x - 1)` |
| Isolate lowest set bit | `x & -x` |
| Is x a power of two? | `x > 0 && (x & (x - 1)) == 0` |
| All ones in the low k bits | `(1 << k) - 1` |
| Iterate all submasks of m | `for (s = m; s; s = (s - 1) & m)` |

```text
x       = 0b101100  (44)
x - 1   = 0b101011
x&(x-1) = 0b101000  lowest 1 removed
-x      = ...010100
x & -x  = 0b000100  only the lowest 1 kept (used by Fenwick trees)
```

### JavaScript: 32-bit signed bitwise
- All bitwise operators convert operands to **32-bit signed integers** (ToInt32), except `>>>`, which produces an **unsigned** 32-bit result. Numbers are 64-bit floats otherwise, so bits above 31 are silently lost.
- `1 << 31` is `-2147483648`. `(1 << 31) >>> 0` is `2147483648`. `2 ** 32 | 0` is `0`.
- `x >>> 0` is the idiom to reinterpret a result as unsigned (reverse bits, hashes).
- `x | 0` and `~~x` truncate toward zero, but only for values within the 32-bit range.
- **BigInt** for wider values: `1n << 40n`, `BigInt.asUintN(64, x)`, `BigInt.asIntN(64, x)`. BigInt supports `& | ^ ~ << >>` but **not `>>>`** (TypeError), and you cannot mix BigInt and Number in one expression.
- `Math.clz32(x)` counts leading zeros in 32 bits. There is no built-in popcount; use the Kernighan loop or a table.

### Python: arbitrary precision
- Python ints never overflow. A negative number behaves as if it had **infinitely many leading 1 bits**, so `-1 & 0xFF == 255` and `bin(-5) == '-0b101'`.
- There is no `>>>`. `>>` is arithmetic. To simulate unsigned 32-bit: `x & 0xFFFFFFFF`.
- To convert a 32-bit unsigned pattern back to signed: `x if x < 2**31 else x - 2**32` (or `~(x ^ 0xFFFFFFFF)`).
- `int.bit_count()` (Python 3.10+) returns the number of 1 bits of the absolute value; `int.bit_length()` returns the number of bits needed. Before 3.10: `bin(x).count("1")`.
- Loops like `while n: n &= n - 1` never terminate for negative `n` in Python (infinite ones). Mask first.

## Complexity

| Operation | Time | Space |
|---|---|---|
| Any single bitwise op on a machine word | O(1) | O(1) |
| Count bits with Kernighan's loop | O(number of set bits) | O(1) |
| `int.bit_count()` / hardware popcount | O(1) for word-size ints | O(1) |
| Counting bits for 0..n (DP) | O(n) | O(n) output |
| Enumerate all subsets via masks | O(2^n · n) | O(n) per subset |
| Enumerate all submasks of all masks | O(3^n) | O(1) |
| Bitmask DP over n items | typically O(2^n · n) or O(2^n · n²) | O(2^n · n) |
| Python ops on big ints (b bits) | O(b) | O(b) |

## Template

Two templates cover most problems: **XOR accumulate** (pairs cancel) and **clear lowest bit in a loop**.

```ts
// XOR accumulate: every value appearing twice cancels out.
function xorAll(nums: number[]): number {
  let acc = 0;
  for (const x of nums) acc ^= x;
  return acc;
}

// Kernighan: count set bits by clearing the lowest one each step.
function popcount32(n: number): number {
  let count = 0;
  while (n !== 0) {
    n &= n - 1;          // works for "unsigned" inputs too: ToInt32 wraps them
    count++;
  }
  return count;
}

// Enumerate subsets with a bitmask (n up to about 20).
function subsetsByMask<T>(items: T[]): T[][] {
  const n = items.length, res: T[][] = [];
  for (let mask = 0; mask < (1 << n); mask++) {
    const subset: T[] = [];
    for (let i = 0; i < n; i++) if ((mask >> i) & 1) subset.push(items[i]);
    res.push(subset);
  }
  return res;
}
```

```python
from functools import reduce
from operator import xor

def xor_all(nums: list[int]) -> int:
    return reduce(xor, nums, 0)

def popcount(n: int) -> int:
    return (n & 0xFFFFFFFF).bit_count()      # Python 3.10+; mask handles negatives

def subsets_by_mask(items: list) -> list[list]:
    n = len(items)
    return [[items[i] for i in range(n) if mask >> i & 1] for mask in range(1 << n)]
```

## Worked example

**[136. Single Number](https://leetcode.com/problems/single-number/)**: every element appears twice except one. Find it in O(n) time and O(1) space.

Idea: XOR everything. Pairs cancel (`a ^ a = 0`), order does not matter, and `0 ^ single = single`.

Input `[4, 1, 2, 1, 2]`:

| step | x | x (binary) | acc before | acc after (binary) | acc after |
|---|---|---|---|---|---|
| 1 | 4 | 100 | 0 (000) | 100 | 4 |
| 2 | 1 | 001 | 4 (100) | 101 | 5 |
| 3 | 2 | 010 | 5 (101) | 111 | 7 |
| 4 | 1 | 001 | 7 (111) | 110 | 6 |
| 5 | 2 | 010 | 6 (110) | 100 | **4** |

Answer **4**. Equivalently: `4 ^ (1 ^ 1) ^ (2 ^ 2) = 4 ^ 0 ^ 0 = 4`.

```ts
function singleNumber(nums: number[]): number {
  let acc = 0;
  for (const x of nums) acc ^= x;
  return acc;
}
```

```python
def single_number(nums: list[int]) -> int:
    acc = 0
    for x in nums:
        acc ^= x
    return acc
```

A hash set also works but costs O(n) space; the XOR trick is the expected answer.

## More templates

### Counting bits for 0..n ([338](https://leetcode.com/problems/counting-bits/))

`i >> 1` drops the last bit, which is `i & 1`. So `bits[i] = bits[i >> 1] + (i & 1)`. For n = 5: `[0, 1, 1, 2, 1, 2]`.

```ts
function countBits(n: number): number[] {
  const bits = new Array<number>(n + 1).fill(0);
  for (let i = 1; i <= n; i++) bits[i] = bits[i >> 1] + (i & 1);
  return bits;
}
```

```python
def count_bits(n: int) -> list[int]:
    bits = [0] * (n + 1)
    for i in range(1, n + 1):
        bits[i] = bits[i >> 1] + (i & 1)
    return bits
```

### Missing number ([268](https://leetcode.com/problems/missing-number/))

XOR all indices `0..n` with all values; everything present cancels, the missing one remains.

```ts
function missingNumber(nums: number[]): number {
  let acc = nums.length;
  for (let i = 0; i < nums.length; i++) acc ^= i ^ nums[i];
  return acc;
}
```

```python
def missing_number(nums: list[int]) -> int:
    acc = len(nums)
    for i, x in enumerate(nums):
        acc ^= i ^ x
    return acc
```

### Reverse bits, unsigned 32-bit ([190](https://leetcode.com/problems/reverse-bits/))

```ts
function reverseBits(n: number): number {
  let res = 0;
  for (let i = 0; i < 32; i++) {
    res = (res << 1) | (n & 1);
    n >>>= 1;               // logical shift: shift in zeros
  }
  return res >>> 0;         // reinterpret as unsigned
}
```

```python
def reverse_bits(n: int) -> int:
    res = 0
    for _ in range(32):
        res = (res << 1) | (n & 1)
        n >>= 1
    return res              # Python ints never go negative here
```

### Add without `+` ([371. Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/))

XOR adds without carry; AND then shift gives the carry. Repeat until no carry. This is where JS and Python differ the most.

```ts
function getSum(a: number, b: number): number {
  while (b !== 0) {
    const carry = (a & b) << 1;   // 32-bit wraparound is automatic in JS
    a = a ^ b;
    b = carry;
  }
  return a;
}
```

```python
def get_sum(a: int, b: int) -> int:
    MASK, MAX_INT = 0xFFFFFFFF, 0x7FFFFFFF
    while b != 0:
        # Mask to 32 bits, otherwise a negative carry grows forever.
        a, b = (a ^ b) & MASK, ((a & b) << 1) & MASK
    return a if a <= MAX_INT else ~(a ^ MASK)   # back to signed
```

### Bitmask DP (mention)
When n is small (about 20 or less) and the state is "which items have been used", encode that set as a mask: `dp[mask][i]` = best cost having visited exactly `mask` and ending at `i`. Classic uses: traveling salesman in O(2^n · n²), assignment problems, and BFS over `(node, mask)` states as in [847. Shortest Path Visiting All Nodes](https://leetcode.com/problems/shortest-path-visiting-all-nodes/) (Hard). See [[Dynamic Programming]].

## When to use it (recognition signals)

- **"Every element appears twice/three times except ..."**: XOR or per-bit counting.
- **"Without using + or -"**, "power of two", "number of 1 bits", "reverse bits", "hamming distance".
- **O(1) extra space** required where a hash set would be the easy answer.
- **Small n (up to about 20) with subsets** of items: masks for enumeration, bitmask DP for optimization.
- **Flags and compact sets**: visited states in BFS, letters present in a word (26-bit mask), permissions.
- Constraints mention **32-bit signed/unsigned integers** explicitly: think about overflow and language semantics.

## Trade-offs & pitfalls

- **Operator precedence.** In JS, C and Java, `&`, `^`, `|` bind **looser** than `==`: `x & 1 == 0` means `x & (1 == 0)`. Always parenthesize: `(x & 1) === 0`. (In Python, comparisons bind looser than `&`, so it happens to work, but parenthesize anyway for readers.)
- **JS 32-bit truncation.** Bitwise ops on numbers beyond 2^31 give wrong results silently. `(2 ** 32 + 5) | 0 === 5`. Use BigInt or arithmetic for large values.
- **Signed vs unsigned in JS.** Results with bit 31 set come out negative. Apply `>>> 0` before returning an unsigned value.
- **`1 << 31` and `1 << 32` in JS.** The shift count is taken modulo 32, so `1 << 32 === 1`, not 0.
- **Python negatives.** Infinite leading ones make loops like `while n` run forever on negative numbers, and `>>` never reaches 0 for negatives (it reaches -1). Mask with `0xFFFFFFFF` first.
- **Readability.** Clever bit hacks are hard to review. Wrap them in named helpers (`isPowerOfTwo`, `lowestSetBit`) and comment the identity used.
- **Right shift of negatives** is arithmetic (`-8 >> 1 === -4`) and rounds toward negative infinity (`-7 >> 1 === -4`), unlike `Math.trunc(-7 / 2) === -3`.

## Senior perspective

- **Bits are how real systems save memory and time**: permission flags (Unix `rwx` modes like `0o755`), feature flags packed in an integer, React's internal lane priorities are bitmasks, TCP flags, IPv4 subnet masks (`ip & mask` to test CIDR membership), and file format headers.
- **Bitsets and bitmaps** power databases (bitmap indexes, Roaring bitmaps in Lucene, Druid and ClickHouse), Bloom filters, Redis `SETBIT` / `BITCOUNT`, and garbage collector mark bits (see [[Memory Management & Garbage Collection]]).
- **Hardware support**: modern CPUs have `POPCNT`, `LZCNT`, `TZCNT` instructions; compilers and runtimes map `bit_count`, `Math.clz32`, `Integer.bitCount` to them. Fenwick trees rely on `x & -x`.
- **Hashing and randomness**: hash functions (FNV, MurmurHash, xorshift PRNGs) are shifts and XORs; in JS they use `Math.imul` and `>>> 0` to stay in 32 bits.
- **Portability**: bit-level code is where language semantics diverge most (JS 32-bit, Python arbitrary precision, Java's fixed widths, C's undefined behavior on signed overflow). In review, ask about width, signedness and endianness.
- **In interviews**, state the identity you rely on (`a ^ a = 0`, `x & (x - 1)`), walk a tiny binary trace, and mention the language caveat (JS `>>> 0`, Python masking). That shows you understand, not memorize.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 136 | [Single Number](https://leetcode.com/problems/single-number/) | Easy | XOR all, pairs cancel |
| 191 | [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) | Easy | `n & (n - 1)` loop or `bit_count()` |
| 338 | [Counting Bits](https://leetcode.com/problems/counting-bits/) | Easy | `bits[i] = bits[i >> 1] + (i & 1)` |
| 268 | [Missing Number](https://leetcode.com/problems/missing-number/) | Easy | XOR indices with values |
| 190 | [Reverse Bits](https://leetcode.com/problems/reverse-bits/) | Easy | shift out and in, `>>> 0` in JS |
| 371 | [Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/) | Medium | XOR plus carry, mask in Python |

Next steps: [137. Single Number II](https://leetcode.com/problems/single-number-ii/) (count bits mod 3), [231. Power of Two](https://leetcode.com/problems/power-of-two/), [78. Subsets](https://leetcode.com/problems/subsets/) via bitmask.

## Interview questions

> [!question]- What does `x & (x - 1)` do, and what can you use it for?
> Subtracting 1 flips the lowest set bit and all zeros below it, so ANDing clears the lowest set bit. Uses: count set bits in O(number of ones) (Kernighan), check power of two (`x > 0 && (x & (x - 1)) == 0`).

> [!question]- Why does XOR solve Single Number?
> XOR is associative and commutative, `a ^ a = 0` and `a ^ 0 = a`. XORing all values cancels every pair regardless of order, leaving the single one. O(n) time, O(1) space.

> [!question]- What is special about bitwise operations in JavaScript?
> Operands are converted to 32-bit signed integers (except `>>>`, which yields unsigned 32-bit). Larger numbers are truncated, results with bit 31 set are negative, and shift counts are taken modulo 32. Use `>>> 0` for unsigned results and BigInt for wider integers (BigInt has no `>>>`).

> [!question]- How do you handle 32-bit overflow semantics in Python?
> Python ints are unbounded, so mask with `0xFFFFFFFF` after each operation to keep 32 bits, and convert back to signed with `x if x <= 0x7FFFFFFF else ~(x ^ 0xFFFFFFFF)`. Without masking, loops relying on overflow (like adding via XOR and carry) never terminate for negatives.

> [!question]- What does `x & -x` return?
> The lowest set bit of x as a power of two (for example 44 = `0b101100` gives 4). In two's complement, `-x = ~x + 1`, which keeps the lowest 1 and flips everything above it. Fenwick (binary indexed) trees use it to jump between ranges.

> [!question]- How can a bitmask represent a subset, and when is that practical?
> Bit i set means item i is included. All subsets of n items are the integers `0 .. 2^n - 1`. It is practical for n up to about 20 (a million masks), and it enables bitmask DP where the state is "which items are used".

> [!question]- How do you count set bits in Python and JS?
> Python 3.10+: `x.bit_count()` (earlier: `bin(x).count("1")`). JS has no built-in popcount: use Kernighan's loop, or a SWAR trick with `Math.imul`, on the 32-bit value.

## Related
- [[DSA Index]]
- [[Backtracking]] (subsets without recursion) · [[Dynamic Programming]] (bitmask DP) · [[Hash Tables]] (the O(n) space alternative)
- [[Arrays & Strings]] · [[Graph BFS-DFS patterns]] (BFS over `(node, mask)` states)
- [[Big-O & Complexity Analysis]] · [[Memory Management & Garbage Collection]]

## Sources
- [MDN: Bitwise operators and unsigned right shift (>>>)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Unsigned_right_shift)
- [MDN: BigInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt)
- [MDN: Math.clz32](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/clz32)
- [Python docs: int.bit_count and bitwise operations on integer types](https://docs.python.org/3/library/stdtypes.html#bitwise-operations-on-integer-types)
- Henry S. Warren Jr., *Hacker's Delight*, 2nd ed., Addison-Wesley 2012
- [cp-algorithms: Bit manipulation](https://cp-algorithms.com/algebra/bit-manipulation.html)
- [Tech Interview Handbook: Binary](https://www.techinterviewhandbook.org/algorithms/binary/)
