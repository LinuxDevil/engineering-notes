---
tags: [dsa, dsa/algorithms]
aliases: [DP, Dynamic Programming, Memoization, Tabulation, Knapsack, LIS, LCS, Edit Distance]
updated: 2026-09-27
---
# Dynamic Programming

> [!tldr] TL;DR
> **Dynamic programming (DP)** solves problems whose answer can be built from answers to **smaller subproblems that repeat** (overlapping subproblems) and where the best overall answer is made of best sub-answers (optimal substructure). You define a **state**, a **transition** (recurrence) and **base cases**, then either recurse with a cache (**top-down memoization**) or fill a table in order (**bottom-up tabulation**). Time is roughly *number of states × work per transition*; space can often be cut to one or two rows.

## The idea

Naive recursion for Fibonacci recomputes `fib(2)` many times (see the recursion tree in [[Recursion & Backtracking]]). DP says: **compute each subproblem once, remember it**.

```text
Naive fib(5): 15 calls           DP fib(5): 6 states
                                 i:    0  1  2  3  4  5
                                 dp:   0  1  1  2  3  5
```

Two conditions make a problem a DP problem:

1. **Overlapping subproblems**: the same smaller problem is needed many times. (Merge sort splits into *disjoint* halves, so it is divide and conquer, not DP.)
2. **Optimal substructure**: an optimal solution contains optimal solutions to its subproblems. (Shortest paths have it; *longest simple path* does not.)

Recognition signals: "count the number of ways", "min/max cost", "is it possible", "longest/shortest subsequence", choices at each step where greedy fails, and constraints like `n ≤ 1000` (suggests `O(n^2)`) or `n ≤ 20` (suggests bitmask `O(2^n · n)`).

## How it works

### The recipe

1. **State**: what parameters identify a subproblem? Write it in words: "`dp[i]` = max money robbing houses `0..i`".
2. **Transition**: how does a state depend on smaller ones? "`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`".
3. **Base cases**: the smallest states you know directly.
4. **Order**: compute states so dependencies are ready (bottom-up), or let recursion handle it (top-down).
5. **Answer**: which state (or max over states) is the result?
6. **Optimize space**: if `dp[i]` only needs the last one or two rows, keep only those.

### Top-down vs bottom-up

| | Top-down (memoization) | Bottom-up (tabulation) |
|---|---|---|
| How | Recursive function + cache | Loops fill a table in dependency order |
| Pros | Natural to write from the recurrence; only visits reachable states | No recursion depth limit; easy to space-optimize; usually faster constants |
| Cons | Stack depth (CPython limit 1000, Node overflow); call overhead | Must figure out the iteration order; computes every state |
| Tools | `functools.cache` / `Map` | arrays, rolling variables |

```mermaid
flowchart LR
  R[Brute-force recursion] -->|add cache| M[Top-down memo]
  M -->|reverse the order, loops| T[Bottom-up table]
  T -->|keep only needed rows| S[Space optimized]
```

A reliable workflow in interviews: write the recursion, memoize it, then convert to bottom-up if asked.

## Complexity

| Family | States | Transition | Time | Space (optimized) |
|---|---|---|---|---|
| 1D (stairs, house robber) | `n` | `O(1)` | `O(n)` | `O(1)` |
| Coin change (amount `A`, `c` coins) | `A` | `O(c)` | `O(A·c)` | `O(A)` |
| 0/1 knapsack (`n` items, capacity `W`) | `n·W` | `O(1)` | `O(n·W)` pseudo-polynomial | `O(W)` |
| 2D grid `m × n` | `m·n` | `O(1)` | `O(m·n)` | `O(n)` |
| LIS | `n` | `O(n)` / binary search | `O(n^2)` / `O(n log n)` | `O(n)` |
| LCS / edit distance | `m·n` | `O(1)` | `O(m·n)` | `O(min(m,n))` |
| Interval DP | `n^2` | `O(n)` | `O(n^3)` | `O(n^2)` |
| Tree DP | `n` nodes | `O(children)` | `O(n)` | `O(h)` stack |
| Bitmask DP | `2^n · n` | `O(n)` | `O(2^n · n^2)` | `O(2^n · n)` |

Knapsack's `O(n·W)` is **pseudo-polynomial**: polynomial in the *value* of `W`, exponential in its bit length. Knapsack is NP-hard in general.

## Implementation

### 1D: Climbing Stairs (LeetCode 70)

State `dp[i]` = ways to reach step `i`. Transition `dp[i] = dp[i-1] + dp[i-2]`. Base `dp[0] = dp[1] = 1`. Only two previous values are needed.

```ts
function climbStairs(n: number): number {
  let prev = 1, cur = 1;                 // dp[0], dp[1]
  for (let i = 2; i <= n; i++) [prev, cur] = [cur, prev + cur];
  return cur;
}
```

```python
def climb_stairs(n: int) -> int:
    prev, cur = 1, 1
    for _ in range(2, n + 1):
        prev, cur = cur, prev + cur
    return cur
```

### 1D: House Robber (LeetCode 198)

Top-down first, then bottom-up with `O(1)` space.

```ts
function robMemo(nums: number[]): number {
  const memo = new Map<number, number>();
  const best = (i: number): number => {           // max from houses 0..i
    if (i < 0) return 0;
    if (memo.has(i)) return memo.get(i)!;
    const v = Math.max(best(i - 1), best(i - 2) + nums[i]);
    memo.set(i, v);
    return v;
  };
  return best(nums.length - 1);
}

function rob(nums: number[]): number {
  let skip = 0, take = 0;                          // dp[i-2], dp[i-1]
  for (const x of nums) [skip, take] = [take, Math.max(take, skip + x)];
  return take;
}
```

```python
from functools import cache

def rob_memo(nums: list[int]) -> int:
    @cache
    def best(i: int) -> int:
        if i < 0:
            return 0
        return max(best(i - 1), best(i - 2) + nums[i])
    return best(len(nums) - 1)

def rob(nums: list[int]) -> int:
    skip = take = 0
    for x in nums:
        skip, take = take, max(take, skip + x)
    return take
```

Trace for `[2, 7, 9, 3, 1]`:

| x | skip (dp[i-2]) | take (dp[i-1]) after |
|---|---|---|
| 2 | 0 | 2 |
| 7 | 2 | 7 |
| 9 | 7 | 11 |
| 3 | 11 | 11 |
| 1 | 11 | 12 |

### 2D grid: Unique Paths (LeetCode 62)

`dp[r][c] = dp[r-1][c] + dp[r][c-1]`; one row suffices because `row[c]` still holds the value from above when we update it.

```ts
function uniquePaths(m: number, n: number): number {
  const row = new Array<number>(n).fill(1);       // first row: 1 way each
  for (let r = 1; r < m; r++)
    for (let c = 1; c < n; c++) row[c] += row[c - 1]; // above + left
  return row[n - 1];
}
```

```python
def unique_paths(m: int, n: int) -> int:
    row = [1] * n
    for _ in range(1, m):
        for c in range(1, n):
            row[c] += row[c - 1]
    return row[-1]
```

### Knapsack: 0/1 vs unbounded

- **0/1** (each item at most once, e.g. Partition Equal Subset Sum 416): iterate capacity **downward** so `dp[w - wt]` still refers to the previous item row.
- **Unbounded** (reuse items, e.g. Coin Change 322): iterate capacity **upward** so the current item can be reused.

```ts
// 0/1: can we pick a subset summing to target? (LeetCode 416)
function canPartition(nums: number[]): boolean {
  const total = nums.reduce((a, b) => a + b, 0);
  if (total % 2) return false;
  const target = total / 2;
  const dp = new Array<boolean>(target + 1).fill(false);
  dp[0] = true;
  for (const x of nums)
    for (let w = target; w >= x; w--) dp[w] ||= dp[w - x];   // downward
  return dp[target];
}

// Unbounded: fewest coins for amount (LeetCode 322)
function coinChange(coins: number[], amount: number): number {
  const dp = new Array<number>(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (const c of coins)
    for (let a = c; a <= amount; a++) dp[a] = Math.min(dp[a], dp[a - c] + 1); // upward
  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

```python
def can_partition(nums: list[int]) -> bool:
    total = sum(nums)
    if total % 2:
        return False
    target = total // 2
    dp = [True] + [False] * target
    for x in nums:
        for w in range(target, x - 1, -1):    # downward
            dp[w] = dp[w] or dp[w - x]
    return dp[target]

def coin_change(coins: list[int], amount: int) -> int:
    INF = float("inf")
    dp = [0] + [INF] * amount
    for c in coins:
        for a in range(c, amount + 1):        # upward
            dp[a] = min(dp[a], dp[a - c] + 1)
    return -1 if dp[amount] == INF else int(dp[amount])
```

Counting combinations (Coin Change II, 518) uses the same loops with `dp[a] += dp[a - c]`. Coins in the **outer** loop count combinations; amount in the outer loop counts ordered permutations (Combination Sum IV, 377).

### Longest Increasing Subsequence (LeetCode 300)

`O(n^2)`: `dp[i]` = LIS ending at `i` = `1 + max(dp[j])` for `j < i` with `nums[j] < nums[i]`.
`O(n log n)` (patience sorting): `tails[k]` = smallest tail of an increasing subsequence of length `k+1`; binary-search where each number goes.

```ts
function lengthOfLIS(nums: number[]): number {
  const tails: number[] = [];
  for (const x of nums) {
    let lo = 0, hi = tails.length;              // first index with tails[i] >= x
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (tails[mid] < x) lo = mid + 1; else hi = mid;
    }
    tails[lo] = x;                               // replace or append
  }
  return tails.length;
}
```

```python
from bisect import bisect_left

def length_of_lis(nums: list[int]) -> int:
    tails: list[int] = []
    for x in nums:
        i = bisect_left(tails, x)
        if i == len(tails):
            tails.append(x)
        else:
            tails[i] = x
    return len(tails)
```

`tails` is not an actual subsequence; it only has the right length. Reconstructing the sequence needs parent pointers. See [[Binary Search]].

### LCS and Edit Distance (two strings)

State `dp[i][j]` over prefixes `a[0..i)` and `b[0..j)`. Extra row/column of zeros for the empty prefix.

- **LCS (1143)**: match → `dp[i-1][j-1] + 1`, else `max(dp[i-1][j], dp[i][j-1])`.
- **Edit distance (72)**: match → `dp[i-1][j-1]`, else `1 + min(insert dp[i][j-1], delete dp[i-1][j], replace dp[i-1][j-1])`. Base: `dp[i][0] = i`, `dp[0][j] = j`.

```ts
function longestCommonSubsequence(a: string, b: string): number {
  const m = a.length, n = b.length;
  const dp = Array.from({ length: m + 1 }, () => new Array<number>(n + 1).fill(0));
  for (let i = 1; i <= m; i++)
    for (let j = 1; j <= n; j++)
      dp[i][j] = a[i - 1] === b[j - 1]
        ? dp[i - 1][j - 1] + 1
        : Math.max(dp[i - 1][j], dp[i][j - 1]);
  return dp[m][n];
}

function minDistance(a: string, b: string): number {
  const m = a.length, n = b.length;
  const dp = Array.from({ length: m + 1 }, (_, i) =>
    Array.from({ length: n + 1 }, (_, j) => (i === 0 ? j : j === 0 ? i : 0)));
  for (let i = 1; i <= m; i++)
    for (let j = 1; j <= n; j++)
      dp[i][j] = a[i - 1] === b[j - 1]
        ? dp[i - 1][j - 1]
        : 1 + Math.min(dp[i][j - 1], dp[i - 1][j], dp[i - 1][j - 1]);
  return dp[m][n];
}
```

```python
def longest_common_subsequence(a: str, b: str) -> int:
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]   # not [[0]*(n+1)]*(m+1)!
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
    return dp[m][n]

def min_distance(a: str, b: str) -> int:
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(dp[i][j - 1], dp[i - 1][j], dp[i - 1][j - 1])
    return dp[m][n]
```

Edit distance table for `"horse" → "ros"` ends at 3 (replace h→r, delete r, delete e).

### Interval DP: Longest Palindromic Subsequence (LeetCode 516)

State `dp[i][j]` over a substring `s[i..j]`. Iterate by **increasing length** (or `i` from right to left) so inner intervals are ready. Harder members of this family (Burst Balloons 312, matrix-chain multiplication) try every split point `k` inside `[i, j]`, giving `O(n^3)`.

```ts
function longestPalindromeSubseq(s: string): number {
  const n = s.length;
  const dp = Array.from({ length: n }, () => new Array<number>(n).fill(0));
  for (let i = n - 1; i >= 0; i--) {
    dp[i][i] = 1;
    for (let j = i + 1; j < n; j++)
      dp[i][j] = s[i] === s[j] ? dp[i + 1][j - 1] + 2 : Math.max(dp[i + 1][j], dp[i][j - 1]);
  }
  return dp[0][n - 1];
}
```

```python
def longest_palindrome_subseq(s: str) -> int:
    n = len(s)
    dp = [[0] * n for _ in range(n)]
    for i in range(n - 1, -1, -1):
        dp[i][i] = 1
        for j in range(i + 1, n):
            if s[i] == s[j]:
                dp[i][j] = dp[i + 1][j - 1] + 2
            else:
                dp[i][j] = max(dp[i + 1][j], dp[i][j - 1])
    return dp[0][n - 1]
```

### DP on trees (brief): House Robber III (LeetCode 337)

Postorder DFS returns a small tuple per node: `(best if we rob this node, best if we skip it)`. Parents combine children's tuples. See [[Tree Traversal - DFS & BFS]].

```ts
type Node = { val: number; left: Node | null; right: Node | null };

function robTree(root: Node | null): number {
  const go = (n: Node | null): [number, number] => {   // [rob, skip]
    if (!n) return [0, 0];
    const [lr, ls] = go(n.left), [rr, rs] = go(n.right);
    return [n.val + ls + rs, Math.max(lr, ls) + Math.max(rr, rs)];
  };
  return Math.max(...go(root));
}
```

```python
def rob_tree(root) -> int:
    def go(n) -> tuple[int, int]:          # (rob, skip)
        if not n:
            return 0, 0
        lr, ls = go(n.left)
        rr, rs = go(n.right)
        return n.val + ls + rs, max(lr, ls) + max(rr, rs)
    return max(go(root))
```

### Bitmask DP (brief)

When `n ≤ ~20`, a subset can be an integer mask, and the state is `dp[mask]` or `dp[mask][last]`. Classic: travelling salesman (Held-Karp), assigning `n` tasks to `n` workers, Partition to K Equal Sum Subsets (698). Transition: for each `i` not in `mask`, go to `mask | (1 << i)`. See [[Bit Manipulation]].

```ts
// Min cost to assign n tasks to n workers; cost[w][t]. dp[mask] = min cost with tasks in mask done by workers 0..popcount-1
function minAssign(cost: number[][]): number {
  const n = cost.length;
  const dp = new Array<number>(1 << n).fill(Infinity);
  dp[0] = 0;
  for (let mask = 0; mask < 1 << n; mask++) {
    if (dp[mask] === Infinity) continue;
    let w = 0; for (let m = mask; m; m &= m - 1) w++;       // popcount = next worker
    if (w === n) continue;
    for (let t = 0; t < n; t++)
      if (!(mask & (1 << t))) dp[mask | (1 << t)] = Math.min(dp[mask | (1 << t)], dp[mask] + cost[w][t]);
  }
  return dp[(1 << n) - 1];
}
```

```python
def min_assign(cost: list[list[int]]) -> int:
    n = len(cost)
    dp = [float("inf")] * (1 << n)
    dp[0] = 0
    for mask in range(1 << n):
        w = mask.bit_count()          # Python 3.10+; bin(mask).count("1") before
        if dp[mask] == float("inf") or w == n:
            continue
        for t in range(n):
            if not mask & (1 << t):
                nxt = mask | (1 << t)
                dp[nxt] = min(dp[nxt], dp[mask] + cost[w][t])
    return int(dp[-1])
```

## Trade-offs & pitfalls

- **Wrong state**: if the transition needs information the state does not hold (e.g. "was the previous house robbed?"), add a dimension. Most DP bugs are state-definition bugs.
- **Off-by-one with prefix tables**: using `dp[i]` for "first `i` characters" means `a[i-1]` is the current char. Pick one convention and write it down.
- **Iteration order in knapsack**: downward = each item once, upward = unlimited reuse. Swapping them silently changes the problem.
- **Python 2D init**: `[[0] * n] * m` aliases the same row `m` times. Use a comprehension.
- **Recursion depth** in top-down: `n = 10^4` overflows CPython's default limit. Convert to bottom-up.
- **Memo keys in JS**: arrays and objects are compared by reference; use `i * (n+1) + j` or a string key, or a 2D array.
- **Sentinels**: `Infinity` in JS is fine, but `Infinity + 1` stays `Infinity` and `-1` sentinels can win a `max`. Be deliberate.
- **Counting overflow**: counts grow exponentially; problems ask for `mod 1e9+7`. In JS, products of two values near `1e9` exceed `2^53`, so use `BigInt` or split multiplication.
- **Greedy vs DP**: coin change with `[1, 3, 4]` and amount 6: greedy gives 4+1+1, DP gives 3+3. Prove greedy or use DP; see [[Greedy]].

## Senior perspective

- **Where DP runs in production**: diff tools (`git diff`, Myers' algorithm is an LCS-style shortest edit script), spell checkers and fuzzy search (Levenshtein distance), DNA alignment (Needleman-Wunsch, Smith-Waterman), speech/HMM decoding (Viterbi), text layout line breaking (Knuth-Plass in TeX), query optimizers choosing join orders (System R style DP over subsets of tables), shortest paths (Bellman-Ford, Floyd-Warshall are DP).
- **Memoization vs caching**: `functools.cache` / a `Map` in a closure is the same idea as an app-level cache, and it has the same problems: unbounded growth, keying on mutable objects, and purity requirements. See [[Closures & Memory Model]] and [[Memory Management & Garbage Collection]].
- **Memory is the usual limit**: a `10^4 × 10^4` table is 100M cells (400 MB as int32, far more as Python lists). Rolling rows or Hirschberg's divide-and-conquer trick gets LCS/alignment to linear space.
- **Cache locality**: bottom-up with row-major loops is much faster than top-down hash-map memo. Typed arrays (`Int32Array`, `Float64Array`) in JS avoid boxing.
- **Interview communication**: say the state in words, give the recurrence, base cases, order, answer location, then complexity as states × transition. Start top-down if it is faster for you, and offer the space optimization as a follow-up. Interviewers care more about a clean recurrence than about a one-liner.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 70 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Easy | Fibonacci-style 1D |
| 198 | [House Robber](https://leetcode.com/problems/house-robber/) | Medium | Take or skip, two rolling vars |
| 62 | [Unique Paths](https://leetcode.com/problems/unique-paths/) | Medium | 2D grid, one row |
| 322 | [Coin Change](https://leetcode.com/problems/coin-change/) | Medium | Unbounded knapsack, min |
| 518 | [Coin Change II](https://leetcode.com/problems/coin-change-ii/) | Medium | Count combinations, coins outer loop |
| 416 | [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) | Medium | 0/1 knapsack, iterate down |
| 300 | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) | Medium | `O(n^2)` DP or patience + binary search |
| 1143 | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | Medium | Two-string prefix table |
| 139 | [Word Break](https://leetcode.com/problems/word-break/) | Medium | `dp[i]` = prefix breakable |
| 72 | [Edit Distance](https://leetcode.com/problems/edit-distance/) | Medium | Insert / delete / replace |
| 516 | [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/) | Medium | Interval DP |
| 337 | [House Robber III](https://leetcode.com/problems/house-robber-iii/) | Medium | Tree DP, (rob, skip) pairs |
| 312 | [Burst Balloons](https://leetcode.com/problems/burst-balloons/) | Hard | Interval DP, last balloon to burst |

## Interview questions

> [!question]- How do you know a problem needs dynamic programming?
> It asks for a count, a min/max, or feasibility over choices; brute force explores overlapping subproblems (the same arguments recur); and optimal answers are built from optimal sub-answers. Greedy counterexamples are another hint. Constraints like `n ≤ 5000` point to `O(n^2)` DP, `n ≤ 20` to bitmask DP.

> [!question]- Top-down memoization vs bottom-up tabulation: which do you prefer?
> Top-down is closer to the recurrence and only computes reachable states, but it pays for recursion and can hit stack limits (CPython 1000, V8 has no TCO). Bottom-up avoids recursion, has better constants and locality, and makes space optimization easy, but you must work out the fill order. I usually write top-down first, then convert.

> [!question]- What are the steps to design a DP solution?
> Define the state in words, write the transition, set base cases, choose a computation order where dependencies come first, identify which state holds the answer, then analyze time as states × transition cost and optimize space if only recent rows are used.

> [!question]- Why does 0/1 knapsack iterate capacity backwards in the 1D version?
> With one array, `dp[w - wt]` must still hold the value from before the current item was considered. Iterating downward guarantees that. Iterating upward lets the current item be used multiple times, which is the unbounded knapsack.

> [!question]- How do you get LIS in O(n log n)?
> Keep `tails`, where `tails[k]` is the smallest possible tail of an increasing subsequence of length `k+1`. For each number, binary-search the first tail `>= x` and replace it (or append if none). `tails` stays sorted, and its length is the answer.

> [!question]- Is O(n·W) knapsack polynomial?
> No, it is pseudo-polynomial: polynomial in the numeric value of `W` but exponential in the number of bits needed to write `W`. 0/1 knapsack is NP-hard; the DP is efficient only when `W` is small.

> [!question]- How would you reduce the memory of LCS or edit distance?
> Each row only depends on the previous row (and the current row to the left), so keep two rows, or one row plus a saved diagonal value: `O(min(m, n))` space. If you also need the actual alignment, Hirschberg's algorithm recovers it in linear space with divide and conquer.

## Related
- [[DSA Index]]
- [[Recursion & Backtracking]] (memoization bridge) · [[Backtracking]] · [[Greedy]] · [[Binary Search]] · [[Bit Manipulation]] · [[Prefix Sum]]
- [[Tree Traversal - DFS & BFS]] (tree DP) · [[Graphs]] (Bellman-Ford, DAG shortest paths)
- [[Big-O & Complexity Analysis]] · [[Problem Solving]] · [[Grokking Algorithms]] (dynamic programming chapter: knapsack, longest common substring)
- [[Closures & Memory Model]] (memoize helpers)

## Sources
- Cormen et al., *Introduction to Algorithms*, 4th ed., MIT Press 2022 (ch. 14 Dynamic Programming)
- Aditya Bhargava, *Grokking Algorithms* (Manning), dynamic programming chapter (knapsack, longest common substring)
- [cp-algorithms: Introduction to Dynamic Programming](https://cp-algorithms.com/dynamic_programming/intro-to-dp.html)
- [cp-algorithms: Longest increasing subsequence](https://cp-algorithms.com/sequences/longest_increasing_subsequence.html)
- [Python docs: functools.cache](https://docs.python.org/3/library/functools.html#functools.cache) · [bisect](https://docs.python.org/3/library/bisect.html)
- [LeetCode Explore: Dynamic Programming card](https://leetcode.com/explore/featured/card/dynamic-programming/)
- [Tech Interview Handbook: Dynamic programming cheatsheet](https://www.techinterviewhandbook.org/algorithms/dynamic-programming/)
- [NeetCode roadmap](https://neetcode.io/roadmap)
