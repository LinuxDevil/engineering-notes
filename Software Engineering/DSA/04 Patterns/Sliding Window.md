---
title: "Sliding Window"
tags: [dsa, dsa/patterns]
aliases: [Sliding window, Window, Fixed Window, Variable Window, At Most K]
updated: 2026-09-27
---
# Sliding Window

<Note>
**TL;DR**

A **sliding window** is a contiguous subarray (or substring) described by two indices, `left` and `right`. You grow it by moving `right`, and shrink it by moving `left` whenever it breaks a constraint, updating the answer incrementally instead of recomputing each subarray. That turns O(n²) "check every subarray" into O(n). Windows are **fixed size** (length `k`) or **variable size** (grow/shrink by a rule); two extra tricks: the number of valid subarrays ending at `right` is `right - left + 1`, and "exactly K" = `atMost(K) - atMost(K - 1)`.
</Note>

## The idea

My original note (from [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course)):

> Sliding window is all about sub-arrays, each subarray is defined by the left bound and the right bound which will make a sub array which is called here "window"

![image 17](/Attachments/image%2017.png)

A brute force looks at all n(n+1)/2 subarrays. The sliding window only moves forward: `right` visits each index once, `left` visits each index at most once, so the whole thing is at most 2n pointer moves.

```text
nums:   [ 3, 1, 2, 7, 4, 2, 1, 1, 5 ]
               L        R
               └────────┘  window = nums[L..R], state = sum/count/map of it
```

It works when the constraint is **monotonic**: if a window is invalid, every bigger window that contains it is also invalid (e.g. sum > k with non-negative numbers, "more than k distinct characters"). Then shrinking from the left is the only way to fix it, and we never need to move `left` backwards.

## How it works

Two flavours:

| | Fixed size `k` | Variable size |
|---|---|---|
| Window length | always `k` | changes |
| Move `right` | every step | every step |
| Move `left` | exactly once per step after the first `k` | in a `while` loop until the window is valid again |
| Typical question | "max/avg sum of a subarray of length k" | "longest/shortest subarray that satisfies X", "number of subarrays that satisfy X" |

State you maintain (all updated in O(1) per move): a running sum, a counter, a frequency [hash map](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables), or a [monotonic deque](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) for window max/min.

### Counting subarrays: `right - left + 1`

When the window `[left, right]` is valid and the constraint is monotonic, every subarray that **ends at `right`** and starts anywhere in `[left, right]` is also valid. There are `right - left + 1` of them. Add that to the answer each step and you count all valid subarrays in O(n).

### Exactly K = at most K minus at most (K - 1)

"Exactly K distinct" is not monotonic (adding elements can move you from 3 to 4, shrinking from 4 to 3). But "at most K" is. So:

```text
count(exactly K) = count(at most K) - count(at most K - 1)
```

Each `atMost` is a normal variable window using the `right - left + 1` count.

## Complexity

| Case | Time | Space |
|---|---|---|
| Fixed window | O(n) | O(1) |
| Variable window, sum/count state | O(n) (each index enters and leaves once) | O(1) |
| Variable window with frequency map | O(n) | O(alphabet) or O(distinct) |
| Exactly K via two `atMost` calls | O(n) | O(distinct) |
| Brute force for comparison | O(n²) or O(n³) | O(1) |

## Template

### Fixed-size window (LC 643 style)

```ts
function maxSumOfK(nums: number[], k: number): number {
  let sum = 0;
  for (let i = 0; i < k; i++) sum += nums[i];
  let best = sum;
  for (let right = k; right < nums.length; right++) {
    sum += nums[right] - nums[right - k]; // add new, drop the one leaving
    best = Math.max(best, sum);
  }
  return best;
}
```

```python
def max_sum_of_k(nums: list[int], k: int) -> int:
    window = sum(nums[:k])
    best = window
    for right in range(k, len(nums)):
        window += nums[right] - nums[right - k]  # add new, drop the one leaving
        best = max(best, window)
    return best
```

### Variable-size window (longest valid window)

```ts
function longestValid(nums: number[], k: number): number {
  let left = 0, curr = 0, best = 0;
  for (let right = 0; right < nums.length; right++) {
    curr += nums[right];                 // 1. add nums[right] to the state
    while (curr > k) {                   // 2. shrink while invalid
      curr -= nums[left];
      left++;
    }
    best = Math.max(best, right - left + 1); // 3. window is valid: update answer
  }
  return best;
}
```

```python
def longest_valid(nums: list[int], k: int) -> int:
    left = curr = best = 0
    for right, x in enumerate(nums):
        curr += x                      # 1. add to state
        while curr > k:                # 2. shrink while invalid
            curr -= nums[left]
            left += 1
        best = max(best, right - left + 1)  # 3. update answer
    return best
```

### Counting valid subarrays (LC 713 Subarray Product Less Than K)

```ts
function numSubarrayProductLessThanK(nums: number[], k: number): number {
  if (k <= 1) return 0; // product of positives is always >= 1
  let left = 0, prod = 1, count = 0;
  for (let right = 0; right < nums.length; right++) {
    prod *= nums[right];
    while (prod >= k) prod /= nums[left++];
    count += right - left + 1; // all subarrays ending at right
  }
  return count;
}
```

```python
def num_subarray_product_less_than_k(nums: list[int], k: int) -> int:
    if k <= 1:
        return 0
    left, prod, count = 0, 1, 0
    for right, x in enumerate(nums):
        prod *= x
        while prod >= k:
            prod //= nums[left]
            left += 1
        count += right - left + 1
    return count
```

### Exactly K (LC 992 Subarrays with K Different Integers)

```ts
function subarraysWithKDistinct(nums: number[], k: number): number {
  const atMost = (k: number): number => {
    const freq = new Map<number, number>();
    let left = 0, count = 0;
    for (let right = 0; right < nums.length; right++) {
      freq.set(nums[right], (freq.get(nums[right]) ?? 0) + 1);
      while (freq.size > k) {
        const c = freq.get(nums[left])! - 1;
        if (c === 0) freq.delete(nums[left]);
        else freq.set(nums[left], c);
        left++;
      }
      count += right - left + 1;
    }
    return count;
  };
  return atMost(k) - atMost(k - 1);
}
```

```python
from collections import defaultdict

def subarrays_with_k_distinct(nums: list[int], k: int) -> int:
    def at_most(k: int) -> int:
        freq: dict[int, int] = defaultdict(int)
        left = count = 0
        for right, x in enumerate(nums):
            freq[x] += 1
            while len(freq) > k:
                freq[nums[left]] -= 1
                if freq[nums[left]] == 0:
                    del freq[nums[left]]
                left += 1
            count += right - left + 1
        return count

    return at_most(k) - at_most(k - 1)
```

## Worked example

**[3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)**: `s = "abcabcbb"`, answer `3`.

Constraint: the window has no repeated character. Monotonic: if a window has a repeat, any bigger window does too.

| right | char | action | left | window | best |
|---|---|---|---|---|---|
| 0 | a | add | 0 | `a` | 1 |
| 1 | b | add | 0 | `ab` | 2 |
| 2 | c | add | 0 | `abc` | 3 |
| 3 | a | `a` repeated → drop `a` | 1 | `bca` | 3 |
| 4 | b | `b` repeated → drop `b` | 2 | `cab` | 3 |
| 5 | c | `c` repeated → drop `c` | 3 | `abc` | 3 |
| 6 | b | `b` repeated → drop `a`, drop `b` | 5 | `cb` | 3 |
| 7 | b | `b` repeated → drop `c`, drop `b` | 7 | `b` | 3 |

`left` moved 7 times in total, `right` 8 times: linear.

```ts
function lengthOfLongestSubstring(s: string): number {
  const seen = new Set<string>();
  let left = 0, best = 0;
  for (let right = 0; right < s.length; right++) {
    while (seen.has(s[right])) {
      seen.delete(s[left]);
      left++;
    }
    seen.add(s[right]);
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

```python
def length_of_longest_substring(s: str) -> int:
    seen: set[str] = set()
    left = best = 0
    for right, ch in enumerate(s):
        while ch in seen:
            seen.remove(s[left])
            left += 1
        seen.add(ch)
        best = max(best, right - left + 1)
    return best
```

Optimisation: store the last index of each char in a map and jump `left = max(left, last[ch] + 1)` directly, avoiding the inner loop (still O(n)).

## When to use it (recognition signals)

- The question says **subarray** or **substring** (contiguous), not subsequence.
- "Longest / shortest / maximum / minimum ... subarray such that ..." or "number of subarrays such that ...".
- A fixed length `k` is given ("every window of size k", "average of k consecutive").
- The constraint is **monotonic** when you extend the window: sum with non-negative numbers, count of distinct/zeros/bad elements, "at most k replacements".
- "Exactly K" → rewrite as `atMost(K) - atMost(K - 1)`.
- **Negative numbers + sum target** break monotonicity: use [Prefix Sum](/Software%20Engineering/DSA/04%20Patterns/Prefix%20Sum) + hash map instead (LC 560).
- Window max/min per position → [monotonic deque](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) (LC 239).

## Trade-offs & pitfalls

- **Negative numbers**: shrinking no longer reduces the sum, so the window logic is wrong. Switch to prefix sums.
- **Off-by-one in length**: window size is `right - left + 1` (both inclusive).
- **Update the answer at the right time**: for "longest", update after shrinking (window valid). For "shortest" (LC 209), update **inside** the shrink loop while the window is still valid.
- **Map cleanup**: delete keys whose count drops to 0, or `map.size` / `len(dict)` overcounts distinct values.
- **Fixed window start**: don't record the answer until the first `k` elements are in.
- **Product windows**: guard `k <= 1`, and watch for overflow in languages with fixed-width ints.
- A `while` inside a `for` does not make it O(n²): `left` only ever increases, so total inner iterations ≤ n (amortised analysis).

## Senior perspective

- **Rate limiters** use sliding windows: the "sliding window log" keeps timestamps in a deque and drops those older than the window; the "sliding window counter" approximates it with two fixed buckets. Trade-off is memory vs accuracy.
- **Streaming metrics**: moving averages, p99 over the last N minutes, and "max latency in the last 60s" (monotonic deque) in monitoring systems.
- **Networking**: TCP flow control uses a sliding window of unacknowledged bytes. Same name, same shape: a range that advances as data is acked.
- **Stream processing** (Flink, Kafka Streams, SQL `OVER (ROWS BETWEEN ...)`) formalise tumbling (fixed, non-overlapping), hopping and sliding windows.
- In review: is the state update O(1)? Recomputing `Math.max(...window)` each step silently brings back O(n·k).
- In interviews, say the invariant ("the window `[left, right]` is always valid after the while loop") and argue amortised O(n).

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 643 | [Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i/) | Easy | Fixed window, running sum |
| 209 | [Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum/) | Medium | Variable window, update answer inside shrink loop |
| 713 | [Subarray Product Less Than K](https://leetcode.com/problems/subarray-product-less-than-k/) | Medium | Count with `right - left + 1` |
| 1004 | [Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii/) | Medium | Window with at most k zeros |
| 76 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | Hard | Frequency map + "chars satisfied" counter |
| 992 | [Subarrays with K Different Integers](https://leetcode.com/problems/subarrays-with-k-different-integers/) | Hard | `atMost(K) - atMost(K - 1)` |

## Interview questions

<Accordion title="Why is a sliding window with an inner while loop still O(n)?">
Each index is added once (when `right` reaches it) and removed at most once (when `left` passes it). Total pointer moves ≤ 2n, so the inner loop's cost is amortised across the outer loop.
</Accordion>

<Accordion title="Why does sliding window fail for &quot;subarray sum equals k&quot; with negative numbers?">
Shrinking is only correct if removing elements can only make the window "more valid". With negatives, removing an element may increase the sum, so you can't decide which pointer to move. Use a [Prefix Sum](/Software%20Engineering/DSA/04%20Patterns/Prefix%20Sum) with a hash map of seen prefix counts (LC 560), O(n).
</Accordion>

<Accordion title="Why does `right - left + 1` count the valid subarrays ending at `right`?">
Once `[left, right]` is valid and the constraint is monotonic, any subarray `[i, right]` with `left ≤ i ≤ right` is a sub-window, so it's valid too. Starting points `left..right` give `right - left + 1` subarrays, and each subarray is counted exactly once (at its end index).
</Accordion>

<Accordion title="How do you count subarrays with exactly K distinct values?">
Exactly K is not monotonic, but at most K is. Count `atMost(K)` and `atMost(K - 1)` with the counting window and subtract.
</Accordion>

<Accordion title="Fixed vs variable window: how do you pick?">
If the length is given (k), it's fixed: add one, remove one each step. If the question asks for longest/shortest/count under a condition, it's variable: grow with `right`, shrink with `left` while invalid.
</Accordion>

<Accordion title="How would you get the maximum of every window of size k in O(n)?">
Keep a deque of indices whose values are decreasing. Pop smaller values from the back before pushing, drop the front when it leaves the window, and the front is the max. See [Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) (LC 239).
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers) · [Prefix Sum](/Software%20Engineering/DSA/04%20Patterns/Prefix%20Sum) · [Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) · [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) · [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis)
- Course: [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course)

## Sources
- [LeetCode's Interview Crash Course: Data Structures and Algorithms](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/) (Arrays and strings, sliding window)
- [Tech Interview Handbook: Array cheatsheet](https://www.techinterviewhandbook.org/algorithms/array/)
- [NeetCode roadmap: Sliding Window](https://neetcode.io/roadmap)
- [cp-algorithms: Minimum stack / minimum queue](https://cp-algorithms.com/data_structures/stack_queue_modification.html) (window min/max)
- [Python docs: collections.defaultdict](https://docs.python.org/3/library/collections.html#collections.defaultdict)
