---
title: "Prefix Sum"
tags: [dsa, dsa/patterns]
aliases: [Prefix Sums, Cumulative Sum, Running Sum, 2D Prefix Sum, Summed-Area Table, Prefix Sum + Hash Map]
updated: 2026-09-27
---
# Prefix Sum

<Note>
**TL;DR**

A **prefix sum** array stores, at each position, the sum of everything before it. After an O(n) precomputation, the sum of **any** subarray is one subtraction: `prefix[r + 1] - prefix[l]`, O(1). The same idea extends to 2D grids (summed-area tables) and, combined with a **hash map of prefix sums seen so far**, counts subarrays with a target sum in O(n) even with negative numbers.
</Note>

## The idea

If you keep asking "what is the sum of `nums[l..r]`?", summing each time costs O(n) per query. Instead, precompute running totals once:

```text
index:       0   1   2   3   4
nums:      [ 3,  1,  4,  1,  5 ]
prefix: [0,  3,  4,  8,  9, 14 ]     prefix[i] = nums[0] + ... + nums[i-1]

sum(nums[1..3]) = prefix[4] - prefix[1] = 9 - 3 = 6   (1 + 4 + 1)
```

Think of `prefix[i]` as "the odometer reading before index i". The distance travelled between two points is the difference of the readings.

The length `n + 1` version with a leading `0` removes the special case for `l = 0`.

## How it works

| Operation | Formula |
|---|---|
| Build | `prefix[0] = 0`, `prefix[i + 1] = prefix[i] + nums[i]` |
| Range sum `[l, r]` inclusive | `prefix[r + 1] - prefix[l]` |
| Sum of first `k` | `prefix[k]` |
| Sum of the rest after `i` | `prefix[n] - prefix[i + 1]` |

The same shape works for any **invertible** operation: sums (subtract), XOR (XOR again), counts of a property (e.g. number of vowels so far). Products work only if there are no zeros. Max/min are **not** invertible: use a sparse table or segment tree for range max.

### 2D prefix sums (summed-area table)

`P[i][j]` = sum of the rectangle from `(0, 0)` to `(i - 1, j - 1)`. Build with inclusion-exclusion:

```text
P[i+1][j+1] = M[i][j] + P[i][j+1] + P[i+1][j] - P[i][j]
```

Query the rectangle `(r1, c1)` to `(r2, c2)` inclusive:

```text
sum = P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]
```

```text
 ┌─────────┬───────┐
 │    A    │   B   │     region D = total - (A+B) - (A+C) + A
 ├─────────┼───────┤     A was subtracted twice, so add it back once
 │    C    │   D   │
 └─────────┴───────┘
```

### Prefix sum + hash map

`sum(nums[l..r]) = k` ⇔ `prefix[r + 1] - prefix[l] = k` ⇔ `prefix[l] = prefix[r + 1] - k`.

So while scanning, keep a map `count[prefixValue] = how many times seen`. At each position with running sum `s`, the number of subarrays ending here with sum `k` is `count[s - k]`. Seed the map with `{0: 1}` for subarrays that start at index 0. This works with **negative numbers**, unlike [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window).

## Complexity

| Operation | Time | Space |
|---|---|---|
| Build 1D | O(n) | O(n) (O(1) if done in place) |
| Range query 1D | O(1) | |
| Build 2D (m × n) | O(m·n) | O(m·n) |
| Rectangle query 2D | O(1) | |
| Subarray sum equals k (prefix + map) | O(n) | O(n) |
| Point update | O(n) rebuild; use a Fenwick tree for O(log n) | |

## Template

### 1D build and query

```ts
class PrefixSum {
  private readonly prefix: number[];
  constructor(nums: number[]) {
    this.prefix = new Array(nums.length + 1).fill(0);
    for (let i = 0; i < nums.length; i++) this.prefix[i + 1] = this.prefix[i] + nums[i];
  }
  /** sum of nums[l..r], both inclusive */
  sumRange(l: number, r: number): number {
    return this.prefix[r + 1] - this.prefix[l];
  }
}
```

```python
from itertools import accumulate

class PrefixSum:
    def __init__(self, nums: list[int]) -> None:
        self.prefix = [0, *accumulate(nums)]

    def sum_range(self, l: int, r: int) -> int:
        """sum of nums[l..r], both inclusive"""
        return self.prefix[r + 1] - self.prefix[l]
```

### 2D (LC 304 Range Sum Query 2D - Immutable)

```ts
class NumMatrix {
  private readonly P: number[][];
  constructor(matrix: number[][]) {
    const m = matrix.length, n = matrix[0]?.length ?? 0;
    this.P = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
    for (let i = 0; i < m; i++)
      for (let j = 0; j < n; j++)
        this.P[i + 1][j + 1] = matrix[i][j] + this.P[i][j + 1] + this.P[i + 1][j] - this.P[i][j];
  }
  sumRegion(r1: number, c1: number, r2: number, c2: number): number {
    const P = this.P;
    return P[r2 + 1][c2 + 1] - P[r1][c2 + 1] - P[r2 + 1][c1] + P[r1][c1];
  }
}
```

```python
class NumMatrix:
    def __init__(self, matrix: list[list[int]]) -> None:
        m, n = len(matrix), len(matrix[0]) if matrix else 0
        self.P = [[0] * (n + 1) for _ in range(m + 1)]
        for i in range(m):
            for j in range(n):
                self.P[i + 1][j + 1] = (
                    matrix[i][j] + self.P[i][j + 1] + self.P[i + 1][j] - self.P[i][j]
                )

    def sum_region(self, r1: int, c1: int, r2: int, c2: int) -> int:
        P = self.P
        return P[r2 + 1][c2 + 1] - P[r1][c2 + 1] - P[r2 + 1][c1] + P[r1][c1]
```

## Worked example

**[560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/)**: count contiguous subarrays whose sum is `k`. Input: `nums = [1, 2, 3, -2, 2]`, `k = 3`. Answer `4`: `[1,2]`, `[3]`, `[2,3,-2]`, `[3,-2,2]`.

Start with `count = {0: 1}`, `s = 0`, `ans = 0`.

| i | nums[i] | s (running) | need `s - k` | count[need] | ans | map after adding s |
|---|---|---|---|---|---|---|
| 0 | 1 | 1 | -2 | 0 | 0 | \{0:1, 1:1\} |
| 1 | 2 | 3 | 0 | 1 → `[1,2]` | 1 | \{0:1, 1:1, 3:1\} |
| 2 | 3 | 6 | 3 | 1 → `[3]` | 2 | \{…, 6:1\} |
| 3 | -2 | 4 | 1 | 1 → `[2,3,-2]` | 3 | \{…, 4:1\} |
| 4 | 2 | 6 | 3 | 1 → `[3,-2,2]` | 4 | \{…, 6:2\} |

Note the lookup happens **before** inserting the current `s`, otherwise with `k = 0` you'd count an empty subarray.

```ts
function subarraySum(nums: number[], k: number): number {
  const count = new Map<number, number>([[0, 1]]);
  let s = 0, ans = 0;
  for (const x of nums) {
    s += x;
    ans += count.get(s - k) ?? 0;
    count.set(s, (count.get(s) ?? 0) + 1);
  }
  return ans;
}
```

```python
from collections import defaultdict

def subarray_sum(nums: list[int], k: int) -> int:
    count: dict[int, int] = defaultdict(int)
    count[0] = 1
    s = ans = 0
    for x in nums:
        s += x
        ans += count[s - k]
        count[s] += 1
    return ans
```

Variations of the same trick: store the **first index** of each prefix value to get the **longest** subarray (LC 525 Contiguous Array, mapping 0 → -1), or store prefix **mod k** (LC 974).

## When to use it (recognition signals)

- Many **range sum queries** on a static array or grid.
- "Subarray sum equals / divisible by k", especially with **negative numbers** (sliding window won't work).
- "Equal number of 0s and 1s" → map 0 to -1, look for prefix sums that repeat.
- "Pivot / balance index" where left sum equals right sum.
- "Product of array except self" is the multiplicative cousin: prefix products from the left and suffix products from the right.
- Difference arrays (the inverse): many range **updates** `+v on [l, r]`, then read the final array (`diff[l] += v; diff[r+1] -= v`, then prefix-sum `diff`).

## Trade-offs & pitfalls

- **Off-by-one**: pick one convention (length `n + 1` with a leading 0 is the least error-prone) and stick to it.
- **Seed the map with `{0: 1}`** or you miss subarrays starting at index 0.
- **Lookup before insert** in the hash map version.
- **Overflow**: sums of many ints overflow 32-bit in Java/C++; use `long`. JS numbers are exact only up to 2^53 - 1 (use `BigInt` beyond); Python ints are unbounded.
- **Negative modulo**: in JS/Java `-1 % 5 === -1`; normalise with `((s % k) + k) % k` for LC 974. Python's `%` is already non-negative for positive `k`.
- **Mutable arrays**: a point update invalidates O(n) prefix entries. Use a Fenwick (binary indexed) tree or segment tree for mixed updates and queries.
- Prefix sums don't help with range **max/min** (not invertible).

## Senior perspective

- **Summed-area tables / integral images** are the 2D version: the Viola-Jones face detector used them to evaluate box features in O(1), and graphics uses them for fast box blurs.
- **Databases**: `SUM(x) OVER (ORDER BY t ROWS UNBOUNDED PRECEDING)` is a prefix sum; cumulative metrics and running balances in ledgers are the same idea. Precomputed cumulative columns make range aggregates cheap.
- **Time series**: storing cumulative counters (like Prometheus counters) and taking differences between two timestamps is prefix-sum thinking; `rate()` is a difference over a window.
- **Space vs time**: precomputation costs memory and must be rebuilt on writes. For read-heavy static data it's a clear win; for write-heavy data pick a Fenwick tree.
- In review: check the index convention in comments, integer width, and whether the precomputation is actually reused (building a prefix array to answer a single query is wasted work).

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 303 | [Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable/) | Easy | 1D prefix, O(1) query |
| 724 | [Find Pivot Index](https://leetcode.com/problems/find-pivot-index/) | Easy | `left == total - left - nums[i]` |
| 304 | [Range Sum Query 2D - Immutable](https://leetcode.com/problems/range-sum-query-2d-immutable/) | Medium | 2D inclusion-exclusion |
| 525 | [Contiguous Array](https://leetcode.com/problems/contiguous-array/) | Medium | 0 → -1, first index of each prefix |
| 974 | [Subarray Sums Divisible by K](https://leetcode.com/problems/subarray-sums-divisible-by-k/) | Medium | Count prefix sums mod k |
| 238 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Medium | Prefix and suffix products, no division |

## Interview questions

<Accordion title="How do you answer many range-sum queries on a static array?">
Build `prefix` of length n + 1 once in O(n). Each query `[l, r]` is `prefix[r + 1] - prefix[l]` in O(1). Total O(n + q) instead of O(n·q).
</Accordion>

<Accordion title="Why does &quot;subarray sum equals k&quot; need a hash map and not a sliding window?">
With negative numbers the window constraint isn't monotonic. Instead, a subarray `(l, r]` sums to k exactly when `prefix[r] - prefix[l] = k`, so for each `r` you count earlier prefixes equal to `prefix[r] - k` in a hash map. O(n) time, O(n) space.
</Accordion>

<Accordion title="Derive the 2D rectangle query formula.">
Take the big rectangle to `(r2, c2)`, subtract the strip above `r1` and the strip left of `c1`. The top-left corner block was subtracted twice, so add `P[r1][c1]` back once.
</Accordion>

<Accordion title="What if the array is updated between queries?">
A prefix array needs O(n) to update. Use a Fenwick tree (O(log n) update and prefix query, small constant) or a segment tree (also supports range max/min and lazy range updates).
</Accordion>

<Accordion title="What is a difference array?">
The inverse of a prefix sum. To add `v` to every element in `[l, r]`, do `diff[l] += v; diff[r + 1] -= v`. After all updates, one prefix-sum pass over `diff` gives the final array. Many range updates become O(1) each.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window) · [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) · [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings) · [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming) · [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis)
- Course: [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course)

## Sources
- [LeetCode's Interview Crash Course: Data Structures and Algorithms](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/) (Arrays and strings, prefix sum)
- [USACO Guide: Prefix Sums](https://usaco.guide/silver/prefix-sums)
- [cp-algorithms: Fenwick Tree](https://cp-algorithms.com/data_structures/fenwick.html)
- [Python docs: itertools.accumulate](https://docs.python.org/3/library/itertools.html#itertools.accumulate)
- [Tech Interview Handbook: Array cheatsheet](https://www.techinterviewhandbook.org/algorithms/array/)
- Viola & Jones, "Rapid Object Detection using a Boosted Cascade of Simple Features", CVPR 2001 (integral image)
