---
title: "Two Pointers"
tags: [dsa, dsa/patterns]
aliases: [Two Pointer, Two-Pointer Technique, Two pointer technique, Opposite Ends Pointers, Read Write Pointers]
updated: 2026-09-27
---
# Two Pointers

<Note>
**TL;DR**

**Two pointers** walks an array or string with two indices instead of one, so a problem that looks like it needs nested loops (O(n²)) can be solved in a single pass (O(n)) with O(1) extra space. There are three shapes: pointers at **opposite ends** that move toward each other, pointers over **two different sequences** (merge), and **same-direction read/write** pointers (fast reader, slow writer). It almost always relies on some order (sorted input) or a monotonic rule that tells you which pointer to move.
</Note>

## The idea

My original note, from the LeetCode Crash Course "Arrays and strings" chapter ([LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course)):

> There's a couple of ways to implement the two pointers technique:
> 1. Start at the edge of the array, and move until they meet, this will move one pointer from the start and another pointer from the end of the array.

The complete list of variants:

1. **Opposite ends (converging).** `i` starts at index 0, `j` at the last index. Each step you look at `arr[i]` and `arr[j]`, decide, and move one (or both) inward. Stop when they meet.
2. **Two different arrays/strings (merge).** `i` walks `a`, `j` walks `b`. At each step you compare `a[i]` and `b[j]` and advance the pointer whose element you "consumed". This is the merge step of merge sort.
3. **Same direction, read/write (fast/slow index).** `read` scans every element; `write` marks where the next kept element goes. Used to filter or deduplicate **in place**.

```text
1. Opposite ends         2. Two sequences          3. Read / write
 i →           ← j        a: [1 4 7]  i →           [1 1 2 2 3]
[1 2 4 6 8 11]            b: [2 3 9]  j →            w  r →
                          out: [1 2 3 4 7 9]        write only when a new value appears
```

Why it works: every step **discards** at least one candidate (a pair, an element) that can never be part of the answer, so the total work is bounded by the lengths of the inputs.

## How it works

For each variant ask three questions:

| Question | Opposite ends | Two sequences | Read/write |
|---|---|---|---|
| Where do the pointers start? | `0` and `n - 1` | `0` and `0` | `write = 0 or 1`, `read = same` |
| When do they stop? | `i < j` (or `i <= j`) | either one runs out, then drain the rest | `read === n` |
| Which one moves? | the one whose move can improve the answer (decided by sortedness) | the one pointing at the smaller element | `read` always; `write` only when you keep an element |

The "which pointer moves" rule is the whole proof. In Two Sum II (sorted), if `a[i] + a[j]` is too big, no pair using `a[j]` with anything at or right of `i` can work, so `j--` safely throws away all of them.

## Complexity

| Variant | Time | Space |
|---|---|---|
| Opposite ends | O(n) | O(1) |
| Two sequences (merge) | O(n + m) | O(1) extra (O(n + m) if you build an output) |
| Read/write in place | O(n) | O(1) |
| 3Sum (sort + two pointers per anchor) | O(n²) | O(1) extra (ignoring sort) |

If the input is not sorted and you need to sort first, add O(n log n).

## Template

### 1. Opposite ends

```ts
function oppositeEnds(arr: number[]): number {
  let i = 0;
  let j = arr.length - 1;
  let ans = 0;
  while (i < j) {
    // use arr[i], arr[j] to update ans
    if (/* condition to move left */ arr[i] < arr[j]) i++;
    else j--;
  }
  return ans;
}
```

```python
def opposite_ends(arr: list[int]) -> int:
    i, j = 0, len(arr) - 1
    ans = 0
    while i < j:
        # use arr[i], arr[j] to update ans
        if arr[i] < arr[j]:  # condition to move left
            i += 1
        else:
            j -= 1
    return ans
```

### 2. Two sequences (merge)

```ts
function mergeSorted(a: number[], b: number[]): number[] {
  const out: number[] = [];
  let i = 0, j = 0;
  while (i < a.length && j < b.length) {
    if (a[i] <= b[j]) out.push(a[i++]);
    else out.push(b[j++]);
  }
  while (i < a.length) out.push(a[i++]); // drain leftovers
  while (j < b.length) out.push(b[j++]);
  return out;
}
```

```python
def merge_sorted(a: list[int], b: list[int]) -> list[int]:
    out: list[int] = []
    i = j = 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            out.append(a[i]); i += 1
        else:
            out.append(b[j]); j += 1
    out.extend(a[i:])  # drain leftovers
    out.extend(b[j:])
    return out
```

### 3. Same direction read/write (remove duplicates in place, LC 26)

```ts
function removeDuplicates(nums: number[]): number {
  if (nums.length === 0) return 0;
  let write = 1;
  for (let read = 1; read < nums.length; read++) {
    if (nums[read] !== nums[write - 1]) {
      nums[write++] = nums[read];
    }
  }
  return write; // first `write` slots hold the unique values
}
```

```python
def remove_duplicates(nums: list[int]) -> int:
    if not nums:
        return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[write - 1]:
            nums[write] = nums[read]
            write += 1
    return write
```

## Worked example

**[167. Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)**: given a sorted array, return the 1-indexed positions of two numbers that add up to `target`. Must use O(1) extra space.

Input: `numbers = [1, 3, 4, 6, 8, 11]`, `target = 10`.

| Step | i (val) | j (val) | sum | Decision |
|---|---|---|---|---|
| 1 | 0 (1) | 5 (11) | 12 | too big → `j--` (11 can't pair with anything ≥ 1) |
| 2 | 0 (1) | 4 (8) | 9 | too small → `i++` (1 can't pair with anything ≤ 8) |
| 3 | 1 (3) | 4 (8) | 11 | too big → `j--` |
| 4 | 1 (3) | 3 (6) | 9 | too small → `i++` |
| 5 | 2 (4) | 3 (6) | 10 | found → return `[3, 4]` |

Five steps instead of checking all 15 pairs.

```ts
function twoSum(numbers: number[], target: number): number[] {
  let i = 0, j = numbers.length - 1;
  while (i < j) {
    const sum = numbers[i] + numbers[j];
    if (sum === target) return [i + 1, j + 1];
    if (sum > target) j--;
    else i++;
  }
  return [-1, -1]; // problem guarantees a solution
}
```

```python
def two_sum(numbers: list[int], target: int) -> list[int]:
    i, j = 0, len(numbers) - 1
    while i < j:
        s = numbers[i] + numbers[j]
        if s == target:
            return [i + 1, j + 1]
        if s > target:
            j -= 1
        else:
            i += 1
    return [-1, -1]
```

## When to use it (recognition signals)

- Input is **sorted** (or you can sort it without breaking the problem), and you look for a **pair/triplet** with a target sum or difference.
- **Palindromes** or anything symmetric: compare the ends and move inward.
- "Reverse in place", "partition", "move zeroes to the end", "remove duplicates **in place**", "O(1) extra space".
- Two sorted inputs to **merge**, intersect, or check "is `s` a subsequence of `t`".
- "Maximize area/width between two positions" (Container With Most Water).
- If the question is about a **contiguous subarray** with a constraint, you probably want [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window) instead (which is a same-direction two-pointer variant).

## Trade-offs & pitfalls

- **`i < j` vs `i <= j`**: pairs need two distinct elements, so use `i < j`. Palindrome checks also use `i < j` (the middle char matches itself).
- **Sorting destroys indices.** If the problem wants original indices (LC 1 Two Sum, unsorted), sorting + two pointers needs index tracking; a [hash map](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) is simpler.
- **Duplicates in 3Sum/4Sum**: skip equal neighbours after a match and for the anchor, or you return repeated triplets.
- **Forgetting to drain** leftovers in the merge variant.
- **Merging in place** into the first array (LC 88) must go **from the back**, otherwise you overwrite unread values.
- **Moving both pointers** when only one is justified can skip the answer. Only discard what you can prove is useless.
- Strings in JS/Python are immutable; for in-place tricks convert to an array (`[...s]`, `list(s)`).

## Senior perspective

- **Cache friendly**: both pointers scan memory sequentially, which prefetchers love. On large arrays this is often faster in practice than a hash-based O(n) solution with random access.
- **Merge is everywhere**: merge sort, external sorting of files larger than RAM (k-way merge), **sort-merge joins** in databases, and LSM-tree compaction (merging sorted SSTables) are all variant 2 at scale.
- **Read/write compaction** is how `Array.prototype.filter`-like operations are done in place, and how garbage collectors compact live objects (a "sliding" compactor moves live objects down to a write pointer).
- In review, ask: is the input guaranteed sorted? If not, is the O(n log n) sort acceptable, and does it mutate the caller's array (prefer `toSorted()` in modern JS)?
- In interviews, state the invariant out loud ("everything left of `i` and right of `j` has been ruled out"). It turns an intuition into a proof.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 125 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Easy | Opposite ends, skip non-alphanumerics |
| 88 | [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/) | Easy | Merge from the back into `nums1` |
| 283 | [Move Zeroes](https://leetcode.com/problems/move-zeroes/) | Easy | Read/write pointers, then fill zeros |
| 26 | [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/) | Easy | Write only when value changes |
| 11 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Medium | Move the shorter wall inward |
| 15 | [3Sum](https://leetcode.com/problems/3sum/) | Medium | Sort, fix an anchor, two pointers on the rest, skip duplicates |

## Interview questions

<Accordion title="Why is it safe to move the shorter wall in Container With Most Water?">
Area = `min(h[i], h[j]) * (j - i)`. If `h[i]` is the shorter wall, every other container using `i` with a right wall inside `(i, j)` has smaller width and height capped by `h[i]`, so it can't beat the current area. Discarding `i` loses nothing.
</Accordion>

<Accordion title="Two Sum on an unsorted array: two pointers or hash map?">
Hash map: O(n) time, O(n) space, keeps original indices. Two pointers needs a sort first (O(n log n)) and loses indices unless you sort pairs `(value, index)`. Two pointers wins when the input is already sorted and you need O(1) space (LC 167).
</Accordion>

<Accordion title="How do you merge two sorted arrays in place when `nums1` has extra space at the end?">
Start three pointers at the ends: `i = m - 1`, `j = n - 1`, `k = m + n - 1`. Write the larger of `nums1[i]` and `nums2[j]` into `nums1[k]` and move back. Filling from the back never overwrites unread values. Copy the remaining `nums2` elements at the end.
</Accordion>

<Accordion title="What is the complexity of 3Sum and why can't two pointers make it O(n)?">
O(n²): for each of n anchors you run an O(n) two-pointer scan. There are up to O(n²) distinct triplets in the output in the worst case, and the problem is believed to need about quadratic time in general (the "3SUM conjecture").
</Accordion>

<Accordion title="How is the read/write pattern different from the fast/slow pointer on linked lists?">
Read/write moves both indices forward over an array to compact it; the writer lags when elements are dropped. [Fast and Slow Pointer](/Software%20Engineering/DSA/04%20Patterns/Fast%20and%20Slow%20Pointer) moves pointers at different speeds (1 and 2 steps) over a linked structure to find a middle or a cycle.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings) · [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window) · [Fast and Slow Pointer](/Software%20Engineering/DSA/04%20Patterns/Fast%20and%20Slow%20Pointer) · [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search) · [Sorting Algorithms](/Software%20Engineering/DSA/03%20Algorithms/Sorting%20Algorithms) · [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis)
- Course: [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course) (chapter "Arrays and strings")

## Sources
- [LeetCode's Interview Crash Course: Data Structures and Algorithms](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/) (Arrays and strings, two pointers)
- [Tech Interview Handbook: Array cheatsheet](https://www.techinterviewhandbook.org/algorithms/array/)
- [NeetCode roadmap: Two Pointers](https://neetcode.io/roadmap)
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022 (merge procedure, ch. 2)
- Sedgewick & Wayne, [*Algorithms*, 4th ed.](https://algs4.cs.princeton.edu/22mergesort/) (mergesort and merging)
