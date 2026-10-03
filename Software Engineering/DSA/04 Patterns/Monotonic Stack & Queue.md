---
tags: [dsa, dsa/patterns]
aliases: [Monotonic Stack, Monotonic Queue, Monotonic Deque, Next Greater Element, Monotone Stack]
updated: 2026-09-27
---
# Monotonic Stack & Queue

> [!tldr] TL;DR
> A **monotonic stack** is a stack whose elements are kept in sorted order (all increasing or all decreasing) by popping anything that would break the order before pushing. Each pop is the moment you learn an answer: "the element that just arrived is the **next greater (or smaller)** element for everything I popped". A **monotonic deque** applies the same idea to a sliding window, giving the window **max/min in O(1)**. Every element is pushed and popped at most once, so the whole scan is O(n).

## The idea

Brute force for "next greater element" looks right from every index: O(n²). But once a bigger element arrives, smaller elements to its left will never be anyone's answer again ("they're hidden behind it"). So we can **throw them away**, and what remains is always sorted.

```text
nums: [2, 1, 5, 3, 6]      stack holds indices of values still waiting for a "next greater"
                           (values kept DECREASING from bottom to top)

push 2           stack: [2]
push 1           stack: [2, 1]
5 arrives  → pop 1 (next greater of 1 is 5), pop 2 (next greater of 2 is 5), push 5
3 arrives  → push                          stack: [5, 3]
6 arrives  → pop 3 (→ 6), pop 5 (→ 6), push 6
end        → 6 has no next greater (-1)
```

| You want | Keep the stack | Pop while |
|---|---|---|
| Next **greater** element | decreasing | `top < current` |
| Next **smaller** element | increasing | `top > current` |
| Previous greater / smaller | same stacks | after popping, the new top is the previous one |

## How it works

1. Scan left to right (or right to left, both work).
2. For each element, **pop** while the top violates the order. Each popped element gets its answer from the current element.
3. **Push** the current element (usually its **index**, so you can compute distances and look up values).
4. Whatever is left on the stack at the end has no answer (use a default like `-1` or `0`).

### Monotonic deque for sliding window max/min

For the max of every window of size `k`:
- Keep a deque of indices with **decreasing values**.
- Before pushing index `i`, pop from the **back** while `nums[back] <= nums[i]` (they can never be a max again: `i` is bigger and newer).
- Pop from the **front** if it slid out of the window (`front <= i - k`).
- The **front** is the current window's max.

JavaScript has no built-in deque. `Array.prototype.shift()` is O(n) in general, so use an array plus a `head` index (or a ring buffer). Python has `collections.deque` with O(1) `append`, `pop`, `popleft`.

## Complexity

| Operation | Time | Space |
|---|---|---|
| Next greater/smaller for all elements | O(n) amortised (each index pushed and popped once) | O(n) |
| Largest rectangle in histogram | O(n) | O(n) |
| Sliding window max/min | O(n) total, O(1) amortised per step | O(k) |
| Brute force for comparison | O(n²) or O(n·k) | O(1) |

## Template

### Next greater element (indices, default -1)

```ts
function nextGreater(nums: number[]): number[] {
  const ans = new Array<number>(nums.length).fill(-1);
  const stack: number[] = []; // indices, values decreasing
  for (let i = 0; i < nums.length; i++) {
    while (stack.length > 0 && nums[stack[stack.length - 1]] < nums[i]) {
      ans[stack.pop()!] = nums[i];
    }
    stack.push(i);
  }
  return ans;
}
```

```python
def next_greater(nums: list[int]) -> list[int]:
    ans = [-1] * len(nums)
    stack: list[int] = []  # indices, values decreasing
    for i, x in enumerate(nums):
        while stack and nums[stack[-1]] < x:
            ans[stack.pop()] = x
        stack.append(i)
    return ans
```

For a **circular** array (LC 503), loop `i` from `0` to `2n - 1` and use `nums[i % n]`, pushing only during the first pass.

### Sliding window maximum (LC 239) with a monotonic deque

```ts
function maxSlidingWindow(nums: number[], k: number): number[] {
  const dq: number[] = []; // indices; values decreasing from head to tail
  let head = 0;            // dq[head..] is the live deque (avoids O(n) shift())
  const res: number[] = [];
  for (let i = 0; i < nums.length; i++) {
    while (dq.length > head && nums[dq[dq.length - 1]] <= nums[i]) dq.pop();
    dq.push(i);
    if (dq[head] <= i - k) head++;           // front left the window
    if (i >= k - 1) res.push(nums[dq[head]]);
  }
  return res;
}
```

```python
from collections import deque

def max_sliding_window(nums: list[int], k: int) -> list[int]:
    dq: deque[int] = deque()  # indices; values decreasing
    res: list[int] = []
    for i, x in enumerate(nums):
        while dq and nums[dq[-1]] <= x:
            dq.pop()
        dq.append(i)
        if dq[0] <= i - k:        # front left the window
            dq.popleft()
        if i >= k - 1:
            res.append(nums[dq[0]])
    return res
```

(The TS array grows to n entries; fine for interviews. A production version would use a ring buffer of size k.)

### Largest rectangle in histogram (LC 84)

Increasing stack. When a bar is popped, the current index is its first smaller bar on the **right**, and the new top is its first smaller bar on the **left**, so its maximal width is known.

```ts
function largestRectangleArea(heights: number[]): number {
  const stack: number[] = []; // indices, heights increasing
  let best = 0;
  for (let i = 0; i <= heights.length; i++) {
    const h = i === heights.length ? 0 : heights[i]; // sentinel flushes the stack
    while (stack.length > 0 && heights[stack[stack.length - 1]] >= h) {
      const height = heights[stack.pop()!];
      const left = stack.length > 0 ? stack[stack.length - 1] : -1;
      best = Math.max(best, height * (i - left - 1));
    }
    stack.push(i);
  }
  return best;
}
```

```python
def largest_rectangle_area(heights: list[int]) -> int:
    stack: list[int] = []  # indices, heights increasing
    best = 0
    for i in range(len(heights) + 1):
        h = 0 if i == len(heights) else heights[i]  # sentinel
        while stack and heights[stack[-1]] >= h:
            height = heights[stack.pop()]
            left = stack[-1] if stack else -1
            best = max(best, height * (i - left - 1))
        stack.append(i)
    return best
```

Example: `[2, 1, 5, 6, 2, 3]` → `10` (bars 5 and 6, width 2).

## Worked example

**[739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures/)**: for each day, how many days until a warmer temperature (0 if never)?

Input: `[73, 74, 75, 71, 69, 72, 76, 73]` → output `[1, 1, 4, 2, 1, 1, 0, 0]`.

Stack holds indices with decreasing temperatures. Pop while the top is colder than today; the answer for the popped day is `i - popped`.

| i | temp | pops (answer = i - j) | stack after (index:temp) |
|---|---|---|---|
| 0 | 73 | | [0:73] |
| 1 | 74 | 0 → 1 | [1:74] |
| 2 | 75 | 1 → 1 | [2:75] |
| 3 | 71 | | [2:75, 3:71] |
| 4 | 69 | | [2:75, 3:71, 4:69] |
| 5 | 72 | 4 → 1, 3 → 2 | [2:75, 5:72] |
| 6 | 76 | 5 → 1, 2 → 4 | [6:76] |
| 7 | 73 | | [6:76, 7:73] |
| end | | 6, 7 stay 0 | |

8 pushes, 6 pops: linear.

```ts
function dailyTemperatures(temperatures: number[]): number[] {
  const ans = new Array<number>(temperatures.length).fill(0);
  const stack: number[] = [];
  for (let i = 0; i < temperatures.length; i++) {
    while (stack.length > 0 && temperatures[stack[stack.length - 1]] < temperatures[i]) {
      const j = stack.pop()!;
      ans[j] = i - j;
    }
    stack.push(i);
  }
  return ans;
}
```

```python
def daily_temperatures(temperatures: list[int]) -> list[int]:
    ans = [0] * len(temperatures)
    stack: list[int] = []
    for i, t in enumerate(temperatures):
        while stack and temperatures[stack[-1]] < t:
            j = stack.pop()
            ans[j] = i - j
        stack.append(i)
    return ans
```

## When to use it (recognition signals)

- "**Next / previous greater** (or smaller) element", "how many days until warmer", "stock span".
- "For each element, how far can it extend left/right while it's the min/max?" → rectangles in histograms, sum of subarray minimums (LC 907), trapping rain water (LC 42, also solvable with [[Two Pointers]]).
- **Max/min of every window** of size k, or a window constraint on `max - min` (LC 1438) → monotonic deque with a [[Sliding Window]].
- Removing digits/characters to get the smallest lexicographic result (LC 402, 316): a greedy increasing stack.
- A brute force with a nested loop that "looks forward until something bigger/smaller appears".

## Trade-offs & pitfalls

- **Store indices, not values**, when you need distances, widths, or to check whether an element left a window.
- **Strict vs non-strict comparison** (`<` vs `<=`) matters with duplicates: it decides whether equal elements count as "greater" and, in counting problems (LC 907), prevents double counting (use strict on one side, non-strict on the other).
- **Forgetting leftovers**: elements still on the stack at the end need a default answer, or a sentinel (height 0 at the end in LC 84) to flush them.
- **`shift()` in JS** is O(n); a deque built on `shift()` makes LC 239 O(n·k) in the worst case.
- **Window eviction**: check the front against `i - k` using indices; comparing values fails with duplicates.
- Choosing the wrong monotonic direction gives plausible but wrong answers; write the pop condition in words first ("pop while the top is smaller than me").

## Senior perspective

- **Streaming monitoring**: "max latency / max queue depth over the last N seconds" is a monotonic deque over a time window; it avoids re-scanning or a heap with lazy deletes.
- **Finance**: the stock span problem (LC 901) is literally "how many consecutive previous days had a price ≤ today", computed online with a monotonic stack of `(price, span)`.
- **Graphics and geometry**: the largest rectangle in a histogram generalises to the largest rectangle of 1s in a binary matrix (LC 85) by building a histogram per row; monotonic chains also appear in convex hull algorithms (Andrew's monotone chain).
- **Amortised analysis** is the key argument: an individual step can pop many elements, but total pops ≤ total pushes = n. Say that explicitly in interviews.
- Compared to a heap for window max: heap is O(n log n) with lazy deletion; the monotonic deque is O(n) and simpler, but only works for max/min (not k-th largest).

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 496 | [Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/) | Easy | Decreasing stack over `nums2` + hash map |
| 503 | [Next Greater Element II](https://leetcode.com/problems/next-greater-element-ii/) | Medium | Circular: iterate `2n` with `i % n` |
| 739 | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) | Medium | Answer = index distance on pop |
| 901 | [Online Stock Span](https://leetcode.com/problems/online-stock-span/) | Medium | Stack of `(price, span)`, merge spans on pop |
| 84 | [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | Hard | Increasing stack, width from neighbours, sentinel |
| 239 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | Hard | Monotonic deque of indices |

## Interview questions

> [!question]- Why is a monotonic stack O(n) if there's a while loop inside the for loop?
> Each index is pushed once and popped at most once. The total number of pops across the whole run is ≤ n, so the inner loop costs O(n) in total (amortised O(1) per element).

> [!question]- Increasing or decreasing stack for "next greater element"?
> Decreasing (from bottom to top). You pop while the top is smaller than the current element, and the current element is the popped one's next greater. For next smaller, keep it increasing.

> [!question]- How does the monotonic deque give the window maximum in O(1)?
> It only keeps candidates that could still be a max: indices with decreasing values. A new element removes all smaller ones from the back (they're older and smaller, so useless). Expired indices leave from the front. The front is always the max of the current window.

> [!question]- Explain the largest rectangle in histogram in one minute.
> For each bar, the best rectangle using its full height extends left and right until a shorter bar. An increasing stack finds both boundaries: when a bar is popped by a shorter bar at `i`, the right boundary is `i` and the left boundary is the new top. Width = `i - left - 1`. A sentinel 0 at the end flushes the stack. O(n).

> [!question]- Monotonic deque vs heap for sliding window max?
> Deque: O(n) total, O(k) space, simple, but only max/min. Heap: O(n log n) with lazy deletion of expired items, but generalises (k-th largest needs two heaps or an order-statistic structure). See [[Heaps & Priority Queues]].

> [!question]- How do you handle duplicates in "sum of subarray minimums" (LC 907)?
> Each element's contribution is `value × (left count) × (right count)`. With equal values, one side must use strict `<` and the other `<=` so each subarray's minimum is attributed to exactly one index.

## Related
- [[DSA Index]]
- [[Stacks & Queues]] · [[Sliding Window]] · [[Two Pointers]] · [[Heaps & Priority Queues]] · [[Arrays & Strings]]
- [[Big-O & Complexity Analysis]]
- Course: [[LeetCode Crash Course]]

## Sources
- [cp-algorithms: Minimum stack / Minimum queue](https://cp-algorithms.com/data_structures/stack_queue_modification.html)
- [LeetCode's Interview Crash Course: Data Structures and Algorithms](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/) (Stacks and queues, monotonic)
- [Python docs: collections.deque](https://docs.python.org/3/library/collections.html#collections.deque)
- [MDN: Array.prototype.shift()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/shift)
- [NeetCode roadmap: Stack](https://neetcode.io/roadmap)
- [Tech Interview Handbook: Stack cheatsheet](https://www.techinterviewhandbook.org/algorithms/stack/)
