---
title: "Sorting Algorithms"
tags: [dsa, dsa/algorithms]
aliases: [Sorting, Sort Algorithms, Merge Sort, Quick Sort, Quicksort, Heap Sort, Insertion Sort, Counting Sort, Radix Sort, TimSort, Stable Sort]
updated: 2026-09-27
---
# Sorting Algorithms

<Note>
**TL;DR**

Comparison sorts cannot beat `O(n log n)` in the worst case; **merge sort** (stable, `O(n)` extra space), **quick sort** (fast in practice, `O(n^2)` worst case without a good pivot) and **heap sort** (in-place, not stable) all hit it. The simple `O(n^2)` sorts (bubble, selection, insertion) matter for teaching and for tiny or nearly sorted inputs. Non-comparison sorts (counting, radix, bucket) run in linear time when keys are small integers or uniformly spread. In real code use the built-in: both V8 and CPython use **TimSort**, which is stable and adaptive.
</Note>

## The idea

Sorting puts items in order by a key. Two properties matter as much as speed:

- **Stable**: equal keys keep their original relative order. Sort users by name, then stable-sort by country, and names stay alphabetical within each country.
- **In-place**: uses `O(1)` (or `O(log n)` stack) extra memory instead of a copy.

{/* diagram source: /diagrams/diagram-196.mmd */}
![diagram](/diagrams/diagram-196.svg)

**Why `n log n` is a floor for comparison sorts**: there are `n!` possible orderings, each comparison has two outcomes, so a decision tree needs height at least `log2(n!) ≈ n log2 n` to tell them apart.

## How it works

### Basic sorts (`O(n^2)`)

- **Bubble sort**: repeatedly swap adjacent out-of-order pairs; the largest "bubbles" to the end each pass. Stop early if a pass makes no swaps (then best case is `O(n)`). Stable, in-place. Almost never the right choice.
- **Selection sort**: find the minimum of the unsorted part, swap it to the front. Always `O(n^2)` comparisons but only `O(n)` swaps (useful when writes are expensive). Not stable (the long-distance swap breaks ties). [Grokking Algorithms](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/Grokking%20Algorithms) ch. 2 uses it to introduce arrays vs linked lists.
- **Insertion sort**: grow a sorted prefix; insert each new element by shifting larger ones right. Stable, in-place, **adaptive**: `O(n + inversions)`, so `O(n)` on nearly sorted data. That is why TimSort and introsort use it for small runs.

### Merge sort (divide and conquer)

Split in half, sort each half recursively, **merge** two sorted halves with two pointers. Always `O(n log n)`, stable (take from the left on ties), needs an `O(n)` buffer. Great for linked lists (merge needs no random access, `O(1)` extra space) and for external sorting of data larger than memory.

```text
[38 27 43 3 9 82 10]
split  -> [38 27 43 3] [9 82 10]
split  -> [38 27] [43 3] [9 82] [10]
sort   -> [27 38] [3 43] [9 82] [10]
merge  -> [3 27 38 43] [9 10 82]
merge  -> [3 9 10 27 38 43 82]
```

### Quick sort

Pick a **pivot**, **partition** so smaller items go left and larger go right, then recurse on each side. No merge step; work happens in the partition. [Grokking Algorithms](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/Grokking%20Algorithms) ch. 4 covers it as the showcase of divide and conquer.

- **Lomuto partition**: pivot = last element; one index `i` marks the end of the "< pivot" region. Simple, more swaps, degrades to `O(n^2)` when many keys are equal.
- **Hoare partition**: two indices move inward from both ends and swap pairs that are on the wrong side. About 3× fewer swaps than Lomuto on average, handles duplicates better; the pivot does not necessarily end in its final position.
- **Worst case** `O(n^2)`: pivot is always the min or max, e.g. first/last element on already sorted input. Fixes: **random pivot** (expected `O(n log n)` for any input), median-of-three, and **3-way partitioning** (Dutch national flag) for many duplicates.
- **Introsort** (C++ `std::sort`): quicksort that switches to heap sort if recursion gets too deep, and insertion sort for small slices, guaranteeing `O(n log n)`.
- Recurse into the smaller side first (loop on the larger) to cap stack depth at `O(log n)`.

### Heap sort

Build a max-heap in `O(n)` (heapify from the last parent down), then repeatedly swap the root (max) to the end and sift down. `O(n log n)` worst case, `O(1)` extra space, **not stable**, poor cache locality (jumps between `i` and `2i+1`), so slower than quicksort in practice. See [Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues).

### Non-comparison sorts

- **Counting sort**: count occurrences of each key in `[0, k)`, prefix-sum the counts to get positions, place items (iterate right to left to stay stable). `O(n + k)` time and space. Only for small integer key ranges.
- **Radix sort**: stable counting sort digit by digit, least significant digit first (LSD). `O(d · (n + b))` for `d` digits in base `b`. Used for fixed-width integers, strings of equal length, and GPU sorting.
- **Bucket sort**: distribute values into `n` buckets by range, sort each bucket (insertion sort), concatenate. Expected `O(n)` for uniformly distributed floats; `O(n^2)` if everything lands in one bucket.

## Complexity

| Algorithm | Best | Average | Worst | Extra space | Stable | In-place |
|---|---|---|---|---|---|---|
| Bubble (early exit) | `O(n)` | `O(n^2)` | `O(n^2)` | `O(1)` | Yes | Yes |
| Selection | `O(n^2)` | `O(n^2)` | `O(n^2)` | `O(1)` | No | Yes |
| Insertion | `O(n)` | `O(n^2)` | `O(n^2)` | `O(1)` | Yes | Yes |
| Merge | `O(n log n)` | `O(n log n)` | `O(n log n)` | `O(n)` | Yes | No |
| Quick (random pivot) | `O(n log n)` | `O(n log n)` | `O(n^2)` | `O(log n)` stack | No | Yes |
| Heap | `O(n log n)` | `O(n log n)` | `O(n log n)` | `O(1)` | No | Yes |
| TimSort | `O(n)` | `O(n log n)` | `O(n log n)` | `O(n)` | Yes | No |
| Counting | `O(n + k)` | `O(n + k)` | `O(n + k)` | `O(n + k)` | Yes | No |
| Radix (LSD) | `O(d(n + b))` | `O(d(n + b))` | `O(d(n + b))` | `O(n + b)` | Yes | No |
| Bucket | `O(n)` | `O(n)` expected | `O(n^2)` | `O(n)` | Yes* | No |

\* if the per-bucket sort is stable.

## Implementation

### Insertion sort

```ts
function insertionSort(a: number[]): number[] {
  for (let i = 1; i < a.length; i++) {
    const x = a[i];
    let j = i - 1;
    while (j >= 0 && a[j] > x) { a[j + 1] = a[j]; j--; } // shift right
    a[j + 1] = x;
  }
  return a;
}
```

```python
def insertion_sort(a: list[int]) -> list[int]:
    for i in range(1, len(a)):
        x, j = a[i], i - 1
        while j >= 0 and a[j] > x:
            a[j + 1] = a[j]
            j -= 1
        a[j + 1] = x
    return a
```

### Merge sort

```ts
function mergeSort(a: number[]): number[] {
  if (a.length <= 1) return a;
  const mid = a.length >> 1;
  const left = mergeSort(a.slice(0, mid));
  const right = mergeSort(a.slice(mid));
  const out: number[] = [];
  let i = 0, j = 0;
  while (i < left.length && j < right.length) {
    out.push(left[i] <= right[j] ? left[i++] : right[j++]); // <= keeps it stable
  }
  while (i < left.length) out.push(left[i++]);
  while (j < right.length) out.push(right[j++]);
  return out;
}
```

```python
def merge_sort(a: list[int]) -> list[int]:
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    left, right = merge_sort(a[:mid]), merge_sort(a[mid:])
    out, i, j = [], 0, 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:      # <= keeps it stable
            out.append(left[i]); i += 1
        else:
            out.append(right[j]); j += 1
    out.extend(left[i:])
    out.extend(right[j:])
    return out
```

### Quick sort (Lomuto, random pivot)

```ts
function quickSort(a: number[], lo = 0, hi = a.length - 1): number[] {
  while (lo < hi) {
    const p = partition(a, lo, hi);
    // recurse on the smaller side, loop on the larger: O(log n) stack
    if (p - lo < hi - p) { quickSort(a, lo, p - 1); lo = p + 1; }
    else { quickSort(a, p + 1, hi); hi = p - 1; }
  }
  return a;
}

function partition(a: number[], lo: number, hi: number): number {
  const r = lo + Math.floor(Math.random() * (hi - lo + 1));
  [a[r], a[hi]] = [a[hi], a[r]];              // random pivot to the end
  const pivot = a[hi];
  let i = lo;                                  // a[lo..i-1] < pivot
  for (let j = lo; j < hi; j++) {
    if (a[j] < pivot) { [a[i], a[j]] = [a[j], a[i]]; i++; }
  }
  [a[i], a[hi]] = [a[hi], a[i]];              // pivot to its final spot
  return i;
}
```

```python
import random

def quick_sort(a: list[int], lo: int = 0, hi: int | None = None) -> list[int]:
    if hi is None:
        hi = len(a) - 1
    while lo < hi:
        p = partition(a, lo, hi)
        if p - lo < hi - p:
            quick_sort(a, lo, p - 1); lo = p + 1
        else:
            quick_sort(a, p + 1, hi); hi = p - 1
    return a

def partition(a: list[int], lo: int, hi: int) -> int:
    r = random.randint(lo, hi)
    a[r], a[hi] = a[hi], a[r]
    pivot, i = a[hi], lo
    for j in range(lo, hi):
        if a[j] < pivot:
            a[i], a[j] = a[j], a[i]
            i += 1
    a[i], a[hi] = a[hi], a[i]
    return i
```

### Hoare partition (variant)

Recurse on `[lo, p]` and `[p + 1, hi]`: the returned index splits the range, it is not the pivot's final position.

```ts
function hoarePartition(a: number[], lo: number, hi: number): number {
  const pivot = a[lo + ((hi - lo) >> 1)];
  let i = lo - 1, j = hi + 1;
  while (true) {
    do i++; while (a[i] < pivot);
    do j--; while (a[j] > pivot);
    if (i >= j) return j;
    [a[i], a[j]] = [a[j], a[i]];
  }
}
```

```python
def hoare_partition(a: list[int], lo: int, hi: int) -> int:
    pivot = a[(lo + hi) // 2]
    i, j = lo - 1, hi + 1
    while True:
        i += 1
        while a[i] < pivot:
            i += 1
        j -= 1
        while a[j] > pivot:
            j -= 1
        if i >= j:
            return j
        a[i], a[j] = a[j], a[i]
```

### Heap sort

```ts
function heapSort(a: number[]): number[] {
  const n = a.length;
  const sift = (i: number, size: number): void => {
    while (true) {
      let big = i;
      const l = 2 * i + 1, r = l + 1;
      if (l < size && a[l] > a[big]) big = l;
      if (r < size && a[r] > a[big]) big = r;
      if (big === i) return;
      [a[i], a[big]] = [a[big], a[i]];
      i = big;
    }
  };
  for (let i = (n >> 1) - 1; i >= 0; i--) sift(i, n);   // build max-heap, O(n)
  for (let end = n - 1; end > 0; end--) {
    [a[0], a[end]] = [a[end], a[0]];                     // max to the end
    sift(0, end);
  }
  return a;
}
```

```python
def heap_sort(a: list[int]) -> list[int]:
    def sift(i: int, size: int) -> None:
        while True:
            big, l, r = i, 2 * i + 1, 2 * i + 2
            if l < size and a[l] > a[big]:
                big = l
            if r < size and a[r] > a[big]:
                big = r
            if big == i:
                return
            a[i], a[big] = a[big], a[i]
            i = big

    n = len(a)
    for i in range(n // 2 - 1, -1, -1):
        sift(i, n)
    for end in range(n - 1, 0, -1):
        a[0], a[end] = a[end], a[0]
        sift(0, end)
    return a
```

In Python you would normally just use `heapq` (a min-heap) for heap-based work.

### Counting sort (stable, keys in `[0, k)`)

```ts
function countingSort(a: number[], k: number): number[] {
  const count = new Array<number>(k).fill(0);
  for (const x of a) count[x]++;
  for (let v = 1; v < k; v++) count[v] += count[v - 1];   // end positions
  const out = new Array<number>(a.length);
  for (let i = a.length - 1; i >= 0; i--) out[--count[a[i]]] = a[i]; // right to left = stable
  return out;
}
```

```python
def counting_sort(a: list[int], k: int) -> list[int]:
    count = [0] * k
    for x in a:
        count[x] += 1
    for v in range(1, k):
        count[v] += count[v - 1]
    out = [0] * len(a)
    for x in reversed(a):
        count[x] -= 1
        out[count[x]] = x
    return out
```

With plain integers stability does not show; it matters when sorting records by an integer key, which is exactly what radix sort does per digit.

## What the built-ins do

| Runtime | API | Algorithm | Stable |
|---|---|---|---|
| V8 (Chrome, Node, Deno) | `Array.prototype.sort` | TimSort since V8 7.0 / Chrome 70 (2018), written in Torque | Yes; ES2019 made stability a spec requirement |
| JavaScript (ES2023) | `Array.prototype.toSorted` | Same algorithm, returns a new array | Yes |
| CPython | `list.sort`, `sorted` | Timsort since Python 2.3 (2002); merge policy switched to **Powersort** (Munro & Wild) in Python 3.11 | Yes, guaranteed |
| Java | `Arrays.sort` | Dual-pivot quicksort for primitives, TimSort for objects | Objects only |
| C++ | `std::sort` / `std::stable_sort` | Introsort / merge sort | `stable_sort` only |

**TimSort** in one paragraph: scan for natural **runs** (already ascending, or strictly descending which it reverses), extend short runs to a minimum length with binary insertion sort, then merge runs using a stack with rules that keep merges balanced, plus **galloping** (exponential search) when one run keeps winning. Real data is often partly sorted, so it frequently beats `n log n`. Powersort is a different rule for *which* runs to merge next, with provably near-optimal merge cost.

### Custom comparators

```ts
const nums = [10, 9, 1];
nums.sort();                  // [1, 10, 9]  default compares as strings!
nums.sort((a, b) => a - b);   // [1, 9, 10]  numeric ascending

type User = { name: string; age: number };
const users: User[] = [{ name: "Bo", age: 30 }, { name: "Al", age: 30 }];
// multi-key: age desc, then name asc (localeCompare for human text)
const sorted = users.toSorted((a, b) => b.age - a.age || a.name.localeCompare(b.name));
```

```python
from functools import cmp_to_key
from operator import attrgetter

nums = [10, 9, 1]
nums.sort()                                     # [1, 9, 10]
users = [{"name": "Bo", "age": 30}, {"name": "Al", "age": 30}]
by_age_desc_name = sorted(users, key=lambda u: (-u["age"], u["name"]))
# old-style comparator when a key function is awkward (e.g. LeetCode 179):
largest = sorted(["3", "30", "34"], key=cmp_to_key(lambda a, b: (a + b < b + a) - (a + b > b + a)))
```

- JS comparators return negative / zero / positive. Returning a boolean (`a > b`) is a bug: `false` becomes `0` and the sort is inconsistent.
- Comparators must be **consistent** (transitive, `cmp(a,b) = -cmp(b,a)`); `() => Math.random() - 0.5` is not a valid shuffle. Use Fisher-Yates.
- Python prefers a **key function** (computed once per element, then compared as tuples). `a - b` style overflow does not exist in Python or for JS numbers in normal ranges, but it does in Java/C++ with ints.
- `TypedArray.prototype.sort` sorts numerically by default, unlike `Array`.

## Trade-offs & pitfalls

- Default `Array.prototype.sort()` converts to strings. Always pass a comparator for numbers.
- `sort` mutates in place in both languages. Use `toSorted` (ES2023) / `sorted()` when the caller's array must not change (e.g. React state).
- Quicksort with a first/last pivot on sorted input is `O(n^2)` and can blow the recursion stack.
- Lomuto with many duplicates also degrades; use 3-way partitioning.
- Merge sort with `slice` everywhere allocates `O(n log n)` total garbage; a production version uses one shared buffer.
- Counting sort with a huge key range (`k = 10^9`) is worse than `n log n`. Compress coordinates first or use radix.
- Stability matters for multi-key sorts done in passes and for UI tables where users expect ties to keep order.
- Floating-point `NaN` breaks comparator consistency; filter or handle it explicitly.

## Senior perspective

- **Use the library sort** unless you have a measured reason. It is stable, adaptive, battle-tested and implemented in native code. Custom sorts belong in interviews, embedded systems, GPUs, or specialized data.
- **Sort less**: often you need the top k (heap or quickselect, `O(n log k)` / `O(n)` average; see [Top-K & Heaps pattern](/Software%20Engineering/DSA/04%20Patterns/Top-K%20%26%20Heaps%20pattern)), a single order statistic, or grouping (hash map), not a full sort.
- **Databases** sort with external merge sort when the data does not fit in `work_mem`, and avoid sorting entirely when a B-tree index already provides the order. Ask for the `EXPLAIN` plan.
- **Cache locality**: quicksort and merge sort scan memory sequentially; heap sort jumps around. That is why heap sort loses in practice despite the same Big-O.
- **Stability and determinism** matter in distributed systems and UI: pagination with a non-unique sort key produces duplicates or missing rows between pages. Always add a unique tiebreaker (e.g. `id`).
- **Comparator cost**: sorting 1M objects calls the comparator ~20M times. Precompute keys (decorate-sort-undecorate, which Python's `key=` does for you) and avoid `localeCompare` in hot loops; use a shared `Intl.Collator`.
- **Adversarial input**: a deterministic quicksort pivot can be attacked to force `O(n^2)` (algorithmic complexity DoS). Randomization or introsort prevents it.
- **Interview framing**: state stable vs unstable and in-place vs not, explain why quicksort is usually faster than merge sort (in-place, cache friendly, small constants) and why its worst case does not matter with a random pivot.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 88 | [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/) | Easy | Merge step, fill from the back |
| 75 | [Sort Colors](https://leetcode.com/problems/sort-colors/) | Medium | 3-way partition (Dutch national flag) |
| 912 | [Sort an Array](https://leetcode.com/problems/sort-an-array/) | Medium | Implement merge / heap / random-pivot quick sort |
| 148 | [Sort List](https://leetcode.com/problems/sort-list/) | Medium | Merge sort on a linked list |
| 215 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Medium | Quickselect or size-k heap |
| 56 | [Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Medium | Sort by start, then sweep |
| 179 | [Largest Number](https://leetcode.com/problems/largest-number/) | Medium | Custom comparator `a+b` vs `b+a` |
| 164 | [Maximum Gap](https://leetcode.com/problems/maximum-gap/) | Medium | Bucket sort / pigeonhole, `O(n)` |

## Interview questions

<Accordion title="What does it mean for a sort to be stable, and when does it matter?">
Equal keys keep their original relative order. It matters for multi-pass multi-key sorting (sort by secondary key, then stable-sort by primary), for radix sort, and for UIs where rows with equal values should not jump around. Merge sort, insertion sort and TimSort are stable; quick sort, heap sort and selection sort are not.
</Accordion>

<Accordion title="Why is quicksort often faster than merge sort even though its worst case is worse?">
It sorts in place (no `O(n)` buffer), scans memory sequentially (cache friendly), and has a tight inner loop with few moves. With a random pivot the `O(n^2)` case has negligible probability, and introsort removes it entirely.
</Accordion>

<Accordion title="When does quicksort hit O(n^2) and how do you prevent it?">
When partitions are maximally unbalanced: a fixed first/last pivot on sorted or reverse-sorted input, or Lomuto on many equal keys. Prevent with a random pivot or median-of-three, 3-way partitioning for duplicates, and an introsort fallback to heap sort past a depth limit.
</Accordion>

<Accordion title="Can you sort faster than O(n log n)?">
Not with comparisons alone (decision tree lower bound `log2(n!)`). With extra structure yes: counting sort `O(n + k)` for small integer keys, radix sort `O(d(n + b))` for fixed-width keys, bucket sort expected `O(n)` for uniform data.
</Accordion>

<Accordion title="What sorting algorithm does JavaScript's Array.prototype.sort use? And Python's?">
V8 has used TimSort since V8 7.0 / Chrome 70 (2018), and ES2019 requires `Array.prototype.sort` to be stable. CPython has used Timsort since 2.3; since 3.11 its merge policy is Powersort. Both are stable, adaptive hybrids of merge sort and binary insertion sort.
</Accordion>

<Accordion title="Why does `[10, 9, 1].sort()` return `[1, 10, 9]` in JavaScript?">
Without a comparator, elements are converted to strings and compared by UTF-16 code units, so `"10" < "9"`. Pass `(a, b) => a - b`.
</Accordion>

<Accordion title="Which sort would you use for a linked list, and why?">
Merge sort: it needs only sequential access, splits with fast/slow pointers, and merges by relinking nodes, giving `O(n log n)` time and `O(1)` extra space (bottom-up) or `O(log n)` (recursive). Quicksort and heap sort rely on random access.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis) · [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings) · [Linked Lists](/Software%20Engineering/DSA/02%20Data%20Structures/Linked%20Lists) · [Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues)
- [Recursion & Backtracking](/Software%20Engineering/DSA/03%20Algorithms/Recursion%20%26%20Backtracking) (divide and conquer) · [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search) (needs sorted input) · [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers) · [Intervals](/Software%20Engineering/DSA/04%20Patterns/Intervals) · [Top-K & Heaps pattern](/Software%20Engineering/DSA/04%20Patterns/Top-K%20%26%20Heaps%20pattern) · [Greedy](/Software%20Engineering/DSA/04%20Patterns/Greedy)
- [Grokking Algorithms](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/Grokking%20Algorithms) (ch. 2 selection sort, ch. 4 quicksort)

## Sources
- Cormen et al., *Introduction to Algorithms*, 4th ed., MIT Press 2022 (ch. 2, 6, 7, 8)
- Sedgewick & Wayne, *Algorithms*, 4th ed. ([2.3 Quicksort](https://algs4.cs.princeton.edu/23quicksort/), [2.2 Mergesort](https://algs4.cs.princeton.edu/22mergesort/))
- Aditya Bhargava, *Grokking Algorithms* (Manning), ch. 2 and 4
- [V8 blog: Getting things sorted in V8 (2018)](https://v8.dev/blog/array-sort)
- [V8 feature: Stable Array.prototype.sort](https://v8.dev/features/stable-sort)
- [MDN: Array.prototype.sort](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort) · [MDN: toSorted](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted)
- [CPython Objects/listsort.txt (Timsort and Powersort notes)](https://github.com/python/cpython/blob/main/Objects/listsort.txt)
- [bpo-34561: Switch to Munro & Wild "powersort" merge strategy (merged 2021-09-06, Python 3.11)](https://bugs.python.org/issue34561)
- [Python docs: Sorting Techniques](https://docs.python.org/3/howto/sorting.html)
- J. I. Munro, S. Wild, "Nearly-Optimal Mergesorts", ESA 2018 ([arXiv:1805.04154](https://arxiv.org/abs/1805.04154))
- [cp-algorithms: sorting-related articles](https://cp-algorithms.com/)
