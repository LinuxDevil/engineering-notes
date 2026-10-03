---
tags: [dsa, dsa/data-structures]
aliases: [Heap, Heaps, Priority Queue, Binary Heap, Min Heap, Max Heap, heapq, Heap Sort]
updated: 2026-09-27
---
# Heaps & Priority Queues

> [!tldr] TL;DR
> A **priority queue** is an abstract type: insert items, and always remove the one with the highest priority. The usual implementation is a **binary heap**: a complete binary tree stored in a plain array, where every parent is <= its children (min-heap) or >= them (max-heap). You get O(1) peek, O(log n) push/pop, and O(n) build. Python ships `heapq` (a min-heap on lists, with public max-heap functions since 3.14); JavaScript has **no built-in heap**, so you write a small class.

## The idea

A heap is **not** a sorted structure. It only promises that the smallest (min-heap) or largest (max-heap) element is at the root. That weaker promise is cheaper to maintain than full sorting, and it is exactly what you need for "give me the next most urgent thing": task schedulers, Dijkstra, merging sorted streams, top-K.

Two invariants:

1. **Shape**: a *complete* binary tree (every level full except the last, filled left to right). This is what lets it live in an array with no pointers.
2. **Heap order**: parent <= children (min-heap) or parent >= children (max-heap). Siblings have no order between them.

```text
Max-heap                     Min-heap
        100                          1
      /     \                      /   \
    99       80                   3     2
   /  \     /  \                 / \   /
  20  18   70  29               7   4 5
```

## Array layout (my original notes, corrected)

> My original notes, lightly edited:
>
> To get the parent index, take the current index, divide it by 2, then floor it.
> So if we have a tree like this:
> ```text
>         100
>      99      80
>    20  18  70  29
> ```
> it is represented as:
>
> | Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
> |---|---|---|---|---|---|---|---|---|
> | Value | - | 100 | 99 | 80 | 20 | 18 | 70 | 29 |
>
> If we need the parent of 29, we take its index (7), divide by 2 (7 / 2 = 3.5), then floor it to index 3, which is 80.
> To get the children of a node, multiply its index by 2 (left child), and multiply by 2 and add 1 for the right child. So the children of 100 (index 1) are at 1 * 2 = 2 and 1 * 2 + 1 = 3.

**Correction / clarification:** these formulas are right, but only for the **1-indexed** layout, where index 0 is left unused. The example uses that layout, so it checks out: parent of 29 is index 3 (80), children of 100 are 99 and 80.

Most real code (Python `heapq`, Java `PriorityQueue`, the TS class below) uses the **0-indexed** layout, where the root is at index 0 and the formulas shift by one:

| | 1-indexed (index 0 unused) | 0-indexed (root at 0) |
|---|---|---|
| Parent of i | `floor(i / 2)` (`i >> 1`) | `floor((i - 1) / 2)` (`(i - 1) >> 1`) |
| Left child of i | `2i` | `2i + 1` |
| Right child of i | `2i + 1` | `2i + 2` |
| Root | index 1 | index 0 |
| Last internal node (n items) | `floor(n / 2)` | `floor(n / 2) - 1` |

The same tree in both layouts:

| Node | 1-indexed position | 0-indexed position |
|---|---|---|
| 100 | 1 | 0 |
| 99 | 2 | 1 |
| 80 | 3 | 2 |
| 20 | 4 | 3 |
| 18 | 5 | 4 |
| 70 | 6 | 5 |
| 29 | 7 | 6 |

Worked checks in the 0-indexed layout:

- Parent of 29 (index 6): `floor((6 - 1) / 2) = 2`, which is 80. Same answer as before.
- Children of 100 (index 0): `2*0 + 1 = 1` (99) and `2*0 + 2 = 2` (80).
- Children of 80 (index 2): indexes 5 (70) and 6 (29).

> [!warning] Common bug
> Using the 1-indexed formulas (`i / 2`, `2i`, `2i + 1`) on a 0-indexed array. With the root at 0, `2 * 0 = 0` makes the root its own child and the heap silently breaks. Pick one layout and stick to it.

## How it works

### Sift up (after push)

Append the new item at the end (keeps the shape), then swap it with its parent while it violates heap order.

```text
Push 90 into the max-heap [100, 99, 80, 20, 18, 70, 29] (0-indexed)
append at index 7              -> parent (7-1)>>1 = 3 (20). 90 > 20, swap
now at index 3                 -> parent 1 (99). 90 < 99, stop
result [100, 99, 80, 90, 18, 70, 29, 20]
```

### Sift down (after pop)

Save the root, move the last element to the root (keeps the shape), then swap it with its **larger** child (max-heap) or **smaller** child (min-heap) until order holds.

### Heapify in O(n)

To build a heap from an unordered array, sift down every internal node from the last one (`floor(n/2) - 1`) back to the root. It looks like n log n, but most nodes are near the bottom and sift only a little: about n/4 nodes sift 1 level, n/8 sift 2 levels, and so on. The sum `n * sum(k / 2^(k+1))` converges to O(n) (CLRS 6.3). Pushing items one by one is O(n log n).

### Heap sort

1. Build a max-heap in place, O(n).
2. Repeat n - 1 times: swap the root (max) with the last item of the heap region, shrink the region by one, sift the new root down. O(n log n).

In place, O(1) extra space, guaranteed O(n log n), but **not stable** and cache-unfriendly, which is why real libraries prefer quicksort/introsort or Timsort (see [[Sorting Algorithms]]).

## Complexity

| Operation | Time | Space |
|---|---|---|
| Peek min/max | O(1) | O(1) |
| Push (sift up) | O(log n) | O(1) |
| Pop (sift down) | O(log n) | O(1) |
| pushpop / replace | O(log n) | O(1) |
| Heapify (build from array) | O(n) | O(1) in place |
| Build by n pushes | O(n log n) | O(n) |
| Search arbitrary value | O(n) | O(1) |
| Delete / update arbitrary item | O(n) to find, O(log n) to fix. O(log n) total with an index map | O(n) for the map |
| Heap sort | O(n log n) best, avg, worst | O(1) |
| Top-k of n with a size-k heap | O(n log k) | O(k) |

## Implementation

### A generic heap with a comparator (TypeScript)

JavaScript has no built-in heap or priority queue. This class is 0-indexed and takes a comparator, so one class covers min-heap, max-heap and objects.

```ts
class Heap<T> {
  private a: T[] = [];
  // less(x, y) === true means x should be closer to the top than y
  constructor(private less: (x: T, y: T) => boolean) {}

  get size(): number { return this.a.length; }
  peek(): T | undefined { return this.a[0]; }

  push(x: T): void {
    this.a.push(x);
    this.siftUp(this.a.length - 1);
  }

  pop(): T | undefined {
    const a = this.a;
    if (a.length === 0) return undefined;
    const top = a[0];
    const last = a.pop()!;
    if (a.length > 0) {
      a[0] = last;
      this.siftDown(0);
    }
    return top;
  }

  static from<T>(items: T[], less: (x: T, y: T) => boolean): Heap<T> {
    const h = new Heap(less);
    h.a = items.slice();
    for (let i = (h.a.length >> 1) - 1; i >= 0; i--) h.siftDown(i); // O(n) heapify
    return h;
  }

  private siftUp(i: number): void {
    const a = this.a;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (!this.less(a[i], a[p])) break;
      [a[i], a[p]] = [a[p], a[i]];
      i = p;
    }
  }

  private siftDown(i: number): void {
    const a = this.a, n = a.length;
    while (true) {
      const l = 2 * i + 1, r = l + 1;
      let best = i;
      if (l < n && this.less(a[l], a[best])) best = l;
      if (r < n && this.less(a[r], a[best])) best = r;
      if (best === i) return;
      [a[i], a[best]] = [a[best], a[i]];
      i = best;
    }
  }
}

// MinHeap and MaxHeap are just comparators
const minHeap = new Heap<number>((x, y) => x < y);
[5, 1, 8, 3].forEach((x) => minHeap.push(x));
console.log(minHeap.pop(), minHeap.pop()); // 1 3

const maxHeap = Heap.from([20, 100, 18, 99, 29, 80, 70], (x, y) => x > y);
console.log(maxHeap.peek()); // 100

// Objects: tasks by priority, then by insertion order
type Task = { name: string; priority: number; seq: number };
const tasks = new Heap<Task>((x, y) =>
  x.priority !== y.priority ? x.priority < y.priority : x.seq < y.seq,
);
```

### Sift up / sift down by hand (Python)

Useful to understand what `heapq` does, and for interviews that forbid the library.

```python
class MinHeap:
    def __init__(self, items=None):
        self.a = list(items or [])
        for i in range(len(self.a) // 2 - 1, -1, -1):  # O(n) heapify
            self._sift_down(i)

    def __len__(self):
        return len(self.a)

    def peek(self):
        return self.a[0]

    def push(self, x):
        self.a.append(x)
        self._sift_up(len(self.a) - 1)

    def pop(self):
        a = self.a
        top, last = a[0], a.pop()  # IndexError if empty, like heapq
        if a:
            a[0] = last
            self._sift_down(0)
        return top

    def _sift_up(self, i):
        a = self.a
        while i > 0:
            p = (i - 1) // 2
            if a[i] >= a[p]:
                break
            a[i], a[p] = a[p], a[i]
            i = p

    def _sift_down(self, i):
        a, n = self.a, len(self.a)
        while True:
            l, r, best = 2 * i + 1, 2 * i + 2, i
            if l < n and a[l] < a[best]:
                best = l
            if r < n and a[r] < a[best]:
                best = r
            if best == i:
                return
            a[i], a[best] = a[best], a[i]
            i = best


h = MinHeap([5, 1, 8, 3])
print(h.pop(), h.pop())  # 1 3
```

### Python `heapq` (what you use in practice)

`heapq` works on a plain list, is a **min-heap**, and is 0-indexed.

```python
import heapq

nums = [5, 1, 8, 3, 2]
heapq.heapify(nums)              # O(n), in place
heapq.heappush(nums, 0)          # O(log n)
smallest = heapq.heappop(nums)   # 0
top = nums[0]                    # peek: 1
heapq.heappushpop(nums, 4)       # push then pop, faster than two calls
heapq.heapreplace(nums, 7)       # pop then push
heapq.nsmallest(2, [9, 4, 7, 1]) # [1, 4]
heapq.nlargest(2, [9, 4, 7, 1])  # [9, 7]

# Max-heap before 3.14: negate on the way in and out
mx = []
for x in [3, 10, 5]:
    heapq.heappush(mx, -x)
print(-heapq.heappop(mx))        # 10

# Priorities with payloads: tuples compare element by element.
# Add a counter so ties never compare the payloads (which may not be comparable).
import itertools
counter = itertools.count()
pq = []
heapq.heappush(pq, (2, next(counter), {"task": "write"}))
heapq.heappush(pq, (1, next(counter), {"task": "plan"}))
priority, _, task = heapq.heappop(pq)  # (1, 0, {'task': 'plan'})
```

> [!info]+ 🆕 2025–2026
> **Python 3.14 (released 2025-10-07)** added public max-heap functions to `heapq`: `heapify_max`, `heappush_max`, `heappop_max`, `heapreplace_max`, `heappushpop_max`. On 3.14+ you no longer need the negation trick for numbers:
> ```python
> import heapq  # Python 3.14+
> mx = [3, 10, 5]
> heapq.heapify_max(mx)
> heapq.heappush_max(mx, 7)
> print(heapq.heappop_max(mx))  # 10
> ```
> On LeetCode or older runtimes, keep using negation (or tuples with a negated key). Source: [What's new in Python 3.14](https://docs.python.org/3/whatsnew/3.14.html).

### Comparator-based heap in Python

`heapq` has no `key=` parameter. Options: tuples `(key, counter, item)` as above, or give the item a `__lt__`.

```python
from dataclasses import dataclass, field
import heapq

@dataclass(order=True)
class Job:
    priority: int
    name: str = field(compare=False)  # excluded from ordering

jobs = [Job(3, "deploy"), Job(1, "hotfix"), Job(2, "review")]
heapq.heapify(jobs)
print(heapq.heappop(jobs).name)  # hotfix

# Custom ordering via __lt__, e.g. "longest string first, then alphabetical"
class ByLenDesc:
    def __init__(self, s): self.s = s
    def __lt__(self, other):
        return (-len(self.s), self.s) < (-len(other.s), other.s)
```

### Heap sort

```ts
function heapSort(a: number[]): number[] {
  const n = a.length;
  const siftDown = (i: number, size: number) => {
    while (true) {
      const l = 2 * i + 1, r = l + 1;
      let big = i;
      if (l < size && a[l] > a[big]) big = l;
      if (r < size && a[r] > a[big]) big = r;
      if (big === i) return;
      [a[i], a[big]] = [a[big], a[i]];
      i = big;
    }
  };
  for (let i = (n >> 1) - 1; i >= 0; i--) siftDown(i, n); // build max-heap
  for (let end = n - 1; end > 0; end--) {
    [a[0], a[end]] = [a[end], a[0]]; // move max to its final place
    siftDown(0, end);
  }
  return a;
}
console.log(heapSort([20, 100, 18, 99, 29, 80, 70])); // [18, 20, 29, 70, 80, 99, 100]
```

```python
def heap_sort(a: list[int]) -> list[int]:
    n = len(a)

    def sift_down(i: int, size: int) -> None:
        while True:
            l, r, big = 2 * i + 1, 2 * i + 2, i
            if l < size and a[l] > a[big]:
                big = l
            if r < size and a[r] > a[big]:
                big = r
            if big == i:
                return
            a[i], a[big] = a[big], a[i]
            i = big

    for i in range(n // 2 - 1, -1, -1):
        sift_down(i, n)
    for end in range(n - 1, 0, -1):
        a[0], a[end] = a[end], a[0]
        sift_down(0, end)
    return a
```

### Top-K in one glance (LC 215)

Keep a **min-heap of size k**; its root is the k-th largest. O(n log k). Full pattern in [[Top-K & Heaps pattern]].

```ts
function findKthLargest(nums: number[], k: number): number {
  const h = new Heap<number>((x, y) => x < y);
  for (const x of nums) {
    h.push(x);
    if (h.size > k) h.pop();
  }
  return h.peek()!;
}
```

```python
def find_kth_largest(nums: list[int], k: int) -> int:
    h = nums[:k]
    heapq.heapify(h)
    for x in nums[k:]:
        if x > h[0]:
            heapq.heapreplace(h, x)
    return h[0]
```

## Trade-offs & pitfalls

- **1-indexed vs 0-indexed formulas** (see above). The most common heap bug.
- **Heaps are not sorted.** Iterating over `heapq`'s list does not give sorted order; only repeated pops do.
- **Python min-heap only (before 3.14)**: negate numbers for max-heap. Negation does not work for strings; use a wrapper with `__lt__`.
- **Tuple ties**: `(priority, item)` crashes with `TypeError` when priorities tie and items are not comparable (dicts, custom objects). Add a counter.
- **Changing priority in place** breaks the invariant. Either use an index map with decrease-key, or **lazy deletion**: push the new entry and skip stale ones when popped (what most Dijkstra implementations do).
- **JS `Array.prototype.sort` as a priority queue** (re-sorting after each insert) is O(n log n) per operation. Fine for 10 items, terrible for 10^5.
- **Comparator mistakes in TS**: returning a number from a boolean `less` function, or flipping min/max. Test with a quick push/pop sequence.
- **Heap sort is not stable** and has poor cache locality compared with merge sort or quicksort.

## Senior perspective

- **Where heaps run in production**: OS and runtime schedulers (timers are often a min-heap keyed by deadline; libuv, which powers Node.js, keeps timers in a min-heap), Go's `container/heap` for timers and schedulers, Kubernetes and job queues ordering by priority, Dijkstra and A* in routing, k-way merge of sorted runs in databases and external sort, streaming top-K in analytics, rate limiters and TTL caches evicting the soonest-expiring entry.
- **Language support**: Python `heapq` (functions over a list), Java `PriorityQueue` (min-heap, takes a `Comparator`), C++ `std::priority_queue` (max-heap by default), Go `container/heap` (interface you implement), JavaScript none, so production JS code uses a small class or a library.
- **Beyond binary heaps**: d-ary heaps (wider nodes, shallower, better cache behavior; used for timer queues), pairing and Fibonacci heaps (better theoretical decrease-key, rarely faster in practice), indexed priority queues for decrease-key. For distributed priority, you use a broker with priority support or partitioned queues, not one in-memory heap.
- **Memory and locality**: the array layout has no pointers and good locality at the top levels, which is why binary heaps beat pointer-based heaps in real benchmarks.
- **Interview talking points**: say why a heap beats sorting (O(n log k) vs O(n log n), streaming input), know that heapify is O(n), mention the size-k min-heap trick for "k largest", two heaps for running median (LC 295), and lazy deletion when priorities change.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 1046 | [Last Stone Weight](https://leetcode.com/problems/last-stone-weight/) | Easy | Max-heap, pop two, push difference |
| 703 | [Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/) | Easy | Min-heap of size k |
| 215 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Medium | Size-k min-heap, or quickselect |
| 347 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Medium | Count, then heap by frequency (or bucket sort) |
| 973 | [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | Medium | Max-heap of size k on squared distance |
| 912 | [Sort an Array](https://leetcode.com/problems/sort-an-array/) | Medium | Implement heap sort |
| 23 | [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | Hard | Heap of list heads, O(N log k) |
| 295 | [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Hard | Max-heap for low half, min-heap for high half |

## Interview questions

> [!question]- Why is a heap stored in an array, and what are the index formulas?
> A heap is a complete binary tree, so there are no gaps and positions map to indexes directly, with no pointers and good locality. 0-indexed: parent `(i - 1) // 2`, children `2i + 1` and `2i + 2`. 1-indexed (index 0 unused): parent `i // 2`, children `2i` and `2i + 1`.

> [!question]- Why is building a heap O(n) and not O(n log n)?
> Bottom-up heapify sifts down each internal node. Sift-down cost is proportional to the node's height, and most nodes are near the bottom: half the nodes are leaves (cost 0), a quarter have height 1, and so on. The total is n times a converging series, so O(n).

> [!question]- Heap vs sorted array vs BST for a priority queue?
> Sorted array: O(1) min, O(n) insert. Balanced BST: O(log n) everything plus ordered iteration and arbitrary delete, but more memory and worse locality. Binary heap: O(1) peek, O(log n) push/pop, O(n) build, minimal memory. Use the heap when you only need the extreme element.

> [!question]- How do you implement a max-heap with Python's heapq?
> Before Python 3.14, push `-x` and negate on pop, or push `(-key, counter, item)`. On 3.14+, use `heapify_max`, `heappush_max`, `heappop_max` and friends.

> [!question]- How would you find the k largest elements in a stream of 10^9 numbers?
> Keep a min-heap of size k. For each number, if the heap has fewer than k items push it; else if it is larger than the root, replace the root. O(n log k) time, O(k) memory, single pass. The heap holds the answer at the end.

> [!question]- How do you update the priority of an item already in the heap?
> Either keep a map from item to array index and sift up or down after changing it (indexed priority queue, O(log n)), or use lazy deletion: push a new entry and mark or detect the old one as stale when popped. Dijkstra usually uses lazy deletion.

## Related
- [[DSA Index]]
- [[Top-K & Heaps pattern]] · [[Graphs]] (Dijkstra, Prim) · [[Sorting Algorithms]] (heap sort)
- [[Trees & Binary Search Trees]] (complete binary trees) · [[Stacks & Queues]] · [[Arrays & Strings]]
- [[Big-O & Complexity Analysis]]

## Sources
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022: ch. 6 (Heapsort, priority queues)
- [Python docs: heapq](https://docs.python.org/3/library/heapq.html)
- [What's new in Python 3.14 (heapq max-heap functions)](https://docs.python.org/3/whatsnew/3.14.html)
- [Sedgewick & Wayne, Algorithms 4th ed.: 2.4 Priority Queues](https://algs4.cs.princeton.edu/24pq/)
- [Java docs: PriorityQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/PriorityQueue.html)
- [Go docs: container/heap](https://pkg.go.dev/container/heap)
- [libuv design overview (timers)](https://docs.libuv.org/en/v1.x/design.html)
- [Tech Interview Handbook: Heap cheatsheet](https://www.techinterviewhandbook.org/algorithms/heap/)
