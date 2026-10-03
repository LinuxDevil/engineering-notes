---
tags: [dsa, dsa/data-structures]
aliases: [Arrays, Strings, Array, String, Dynamic Array, ArrayList, Vector, StringBuilder]
updated: 2026-09-27
---
# Arrays & Strings

> [!tldr] TL;DR
> An **array** is a block of contiguous memory: reading or writing index `i` is **O(1)**, but inserting or deleting in the middle shifts everything after it, which is **O(n)**. A **dynamic array** (JS `Array`, Python `list`, Java `ArrayList`) grows by reallocating to a bigger block, which makes `push`/`append` **amortized O(1)**. **Strings** are arrays of characters that are **immutable** in JS and Python, so building one with `+=` in a loop can cost O(n²); collect pieces in a list and `join` instead.

## The idea

Picture a row of numbered lockers, all the same size, side by side. If you know the first locker's address and the locker size, locker `i` is at `base + i * size`. That single multiplication is why arrays give **random access in O(1)**.

```text
index:     0     1     2     3     4
         +-----+-----+-----+-----+-----+
memory:  |  7  |  3  |  9  |  1  |  4  |      address(i) = base + i * 8 bytes
         +-----+-----+-----+-----+-----+
         ^ base
```

The price: the lockers are packed. To insert at index 1 you must move items 1..4 one step right. To delete index 1 you move them one step left.

**Strings** are the same idea for characters. The twist in most high-level languages is that strings are **immutable**: `s[0] = 'x'` does nothing in JS and raises `TypeError` in Python. Every "modification" creates a new string.

## How it works

### Static vs dynamic arrays

| | Static array | Dynamic array |
|---|---|---|
| Size | fixed at creation | grows/shrinks |
| Examples | C `int a[10]`, Java `int[]`, JS `Int32Array`, Python `array.array` (fixed type) | JS `Array`, Python `list`, Java `ArrayList`, C++ `std::vector` |
| Append | impossible without copying | amortized O(1) |
| Memory | exactly `n * size` | `capacity >= n`, some slack |

### Amortized append (why doubling works)

A dynamic array keeps a `capacity` bigger than its `length`. When full, it allocates a new block (typically 1.5x to 2x bigger), copies everything, and frees the old one.

```text
cap 1: [a]              push b -> full, copy 1 -> cap 2
cap 2: [a b]            push c -> full, copy 2 -> cap 4
cap 4: [a b c _]        push d -> free slot
cap 4: [a b c d]        push e -> full, copy 4 -> cap 8
```

Pushing `n` items costs `n` writes plus copies of `1 + 2 + 4 + ... + n/2 < n`. Total `< 2n`, so each push is **O(1) amortized**, even though a single push can be O(n). The growth must be **geometric** (multiply), not additive (`+10`); additive growth gives O(n²) total.

- CPython `list` over-allocates by roughly 1/8 plus a small constant (growth factor about 1.125), which is still geometric.
- Java `ArrayList` grows by 1.5x.
- V8 grows JS arrays by about 1.5x plus 16 for fast (packed) elements.

### Strings are immutable (JS and Python)

```ts
let s = "cat";
(s as any)[0] = "b"; // ignored in sloppy mode; TypeError in strict mode (modules, classes)
console.log(s);      // "cat"
```

```python
s = "cat"
s[0] = "b"           # TypeError: 'str' object does not support item assignment
```

Consequences:
- `s += ch` in a loop creates a new string each time: potentially **O(n²)** total. (CPython has an in-place optimization when the refcount is 1, and V8 uses rope/cons-strings, but neither is guaranteed; don't rely on it in interviews.)
- To edit characters, convert to a mutable array (`s.split("")` / `list(s)`), mutate, then `join`.
- Java has `StringBuilder`, C# has `StringBuilder`, Go has `strings.Builder`. In JS and Python the idiom is **push into an array/list, then `join`**.

### Character counting

Many string problems reduce to "count characters". Two options:
- **Hash map** (`Map` / `Counter`): works for any alphabet.
- **Fixed array of 26** (for lowercase English): faster, less memory, index by `code - 'a'.code`. This is O(1) space because 26 is a constant.

### In-place operations

"In-place" means O(1) extra space: you modify the input array directly, usually with [[Two Pointers]] (a read pointer and a write pointer, or left and right pointers moving inward).

## Complexity

| Operation | Time | Space | Notes |
|---|---|---|---|
| Access / update `a[i]` | O(1) | O(1) | address arithmetic |
| Append (`push` / `append`) | O(1) amortized, O(n) worst | O(1) amortized | resize copies |
| Pop from end | O(1) | O(1) | |
| Insert / delete at index `i` | O(n - i) | O(1) | shift elements |
| Insert / delete at front (`unshift`, `shift`, `insert(0, x)`, `pop(0)`) | O(n) | O(1) | use a deque instead |
| Search (unsorted) | O(n) | O(1) | |
| Search (sorted, [[Binary Search]]) | O(log n) | O(1) | |
| Slice / copy of length k | O(k) | O(k) | `slice` copies in JS and Python |
| String concat `a + b` | O(len a + len b) | new string | immutable |
| `join` of k pieces, total length n | O(n) | O(n) | the right way to build strings |
| Sort | O(n log n) | O(n) (TimSort) | [[Sorting Algorithms]] |

## Implementation

### A minimal dynamic array

```ts
class DynamicArray<T> {
  private data: (T | undefined)[];
  private capacity = 1;
  length = 0;

  constructor() {
    this.data = new Array(this.capacity);
  }

  get(i: number): T {
    if (i < 0 || i >= this.length) throw new RangeError("index out of range");
    return this.data[i] as T;
  }

  push(value: T): void {
    if (this.length === this.capacity) this.resize(this.capacity * 2);
    this.data[this.length++] = value;
  }

  pop(): T | undefined {
    if (this.length === 0) return undefined;
    const value = this.data[--this.length];
    this.data[this.length] = undefined; // drop reference for GC
    if (this.length > 0 && this.length === this.capacity / 4) this.resize(this.capacity / 2);
    return value;
  }

  insert(i: number, value: T): void {
    if (i < 0 || i > this.length) throw new RangeError("index out of range");
    if (this.length === this.capacity) this.resize(this.capacity * 2);
    for (let j = this.length; j > i; j--) this.data[j] = this.data[j - 1]; // shift right
    this.data[i] = value;
    this.length++;
  }

  private resize(newCap: number): void {
    const next = new Array<T | undefined>(newCap);
    for (let i = 0; i < this.length; i++) next[i] = this.data[i];
    this.data = next;
    this.capacity = newCap;
  }
}
```

```python
class DynamicArray:
    def __init__(self):
        self._cap = 1
        self._n = 0
        self._data = [None] * self._cap

    def __len__(self):
        return self._n

    def __getitem__(self, i):
        if not 0 <= i < self._n:
            raise IndexError("index out of range")
        return self._data[i]

    def append(self, value):
        if self._n == self._cap:
            self._resize(2 * self._cap)
        self._data[self._n] = value
        self._n += 1

    def pop(self):
        if self._n == 0:
            raise IndexError("pop from empty array")
        self._n -= 1
        value, self._data[self._n] = self._data[self._n], None
        if 0 < self._n == self._cap // 4:
            self._resize(self._cap // 2)
        return value

    def insert(self, i, value):
        if not 0 <= i <= self._n:
            raise IndexError("index out of range")
        if self._n == self._cap:
            self._resize(2 * self._cap)
        for j in range(self._n, i, -1):   # shift right
            self._data[j] = self._data[j - 1]
        self._data[i] = value
        self._n += 1

    def _resize(self, cap):
        new = [None] * cap
        new[: self._n] = self._data[: self._n]
        self._data, self._cap = new, cap
```

Shrink at 1/4 full (not 1/2) to avoid **thrashing**: alternating push/pop at the boundary would otherwise resize every time.

### Building strings: join, not +=

```ts
function repeatChars(s: string): string {
  const parts: string[] = [];
  for (const ch of s) parts.push(ch, ch);   // collect
  return parts.join("");                    // one O(n) build
}
```

```python
def repeat_chars(s: str) -> str:
    parts = []
    for ch in s:
        parts.append(ch * 2)
    return "".join(parts)
```

### Character counting (26-slot array)

```ts
// LeetCode 242: Valid Anagram
function isAnagram(s: string, t: string): boolean {
  if (s.length !== t.length) return false;
  const count = new Array<number>(26).fill(0);
  const a = "a".charCodeAt(0);
  for (let i = 0; i < s.length; i++) {
    count[s.charCodeAt(i) - a]++;
    count[t.charCodeAt(i) - a]--;
  }
  return count.every((c) => c === 0);
}
```

```python
from collections import Counter

def is_anagram(s: str, t: str) -> bool:
    return len(s) == len(t) and Counter(s) == Counter(t)

def is_anagram_array(s: str, t: str) -> bool:
    if len(s) != len(t):
        return False
    count = [0] * 26
    for a, b in zip(s, t):
        count[ord(a) - ord("a")] += 1
        count[ord(b) - ord("a")] -= 1
    return all(c == 0 for c in count)
```

### In-place: reverse and move zeroes

```ts
// LeetCode 344: Reverse String (in place)
function reverseString(s: string[]): void {
  let l = 0, r = s.length - 1;
  while (l < r) {
    [s[l], s[r]] = [s[r], s[l]];
    l++; r--;
  }
}

// LeetCode 283: Move Zeroes, keeping order of non-zeros
function moveZeroes(nums: number[]): void {
  let write = 0;
  for (let read = 0; read < nums.length; read++) {
    if (nums[read] !== 0) {
      [nums[write], nums[read]] = [nums[read], nums[write]];
      write++;
    }
  }
}
```

```python
def reverse_string(s: list[str]) -> None:
    l, r = 0, len(s) - 1
    while l < r:
        s[l], s[r] = s[r], s[l]
        l += 1
        r -= 1

def move_zeroes(nums: list[int]) -> None:
    write = 0
    for read in range(len(nums)):
        if nums[read] != 0:
            nums[write], nums[read] = nums[read], nums[write]
            write += 1
```

### Rotate an array in place (reverse trick)

Rotating right by `k` = reverse all, reverse first `k`, reverse the rest. O(n) time, O(1) space.

```ts
// LeetCode 189: Rotate Array
function rotate(nums: number[], k: number): void {
  const n = nums.length;
  k %= n;
  const rev = (l: number, r: number) => {
    while (l < r) { [nums[l], nums[r]] = [nums[r], nums[l]]; l++; r--; }
  };
  rev(0, n - 1);
  rev(0, k - 1);
  rev(k, n - 1);
}
```

```python
def rotate(nums: list[int], k: int) -> None:
    n = len(nums)
    k %= n
    def rev(l: int, r: int) -> None:
        while l < r:
            nums[l], nums[r] = nums[r], nums[l]
            l += 1
            r -= 1
    rev(0, n - 1)
    rev(0, k - 1)
    rev(k, n - 1)
```

## Trade-offs & pitfalls

- **Off-by-one**: `range(n)` is `0..n-1`; `slice(a, b)` excludes `b`. Decide on inclusive or exclusive bounds and stick to it.
- **Shifting from the front** (`shift`, `unshift`, `pop(0)`, `insert(0, x)`) is O(n). Inside a loop it becomes O(n²). Use an index pointer or a deque ([[Stacks & Queues]]).
- **Slices copy** in both JS and Python. Passing `nums[1:]` into recursion makes it O(n²); pass indexes instead.
- **Mutation while iterating**: removing from a list while looping over it skips elements. Iterate backwards or build a new list.
- **Aliasing**: `[[0] * 3] * 3` in Python creates three references to the same inner list. Use `[[0] * 3 for _ in range(3)]`. In JS, `new Array(3).fill([])` has the same bug; use `Array.from({ length: 3 }, () => [])`.
- **JS sort is lexicographic by default**: `[10, 9, 1].sort()` gives `[1, 10, 9]`. Always pass `(a, b) => a - b`.
- **Unicode**: JS strings are UTF-16. `"😀".length === 2`, and `s[i]` can return half a surrogate pair. Iterate with `for...of` or `Array.from(s)` to get code points. Python `str` is indexed by code point, but a "character" a user sees (grapheme cluster) can still be several code points.
- **Holes and mixed types in JS arrays** (`arr[1000] = 1` on an empty array, or mixing ints and objects) push V8 into slower element kinds.
- **Integer overflow** is not a JS/Python issue for small sums, but JS numbers lose precision above `2**53 - 1`; use `BigInt` if needed.

## Senior perspective

- **Cache locality is the hidden superpower.** Arrays are contiguous, so iterating one is CPU-cache and prefetcher friendly. A linked list with the same Big-O can be several times slower in practice. This is why "just use an array" wins so often, and why databases, columnar stores and game engines lay data out as arrays (struct-of-arrays).
- **V8 element kinds**: V8 tracks whether an array is `PACKED_SMI_ELEMENTS` (small ints), `PACKED_DOUBLE_ELEMENTS`, or `PACKED_ELEMENTS` (anything), and whether it is `HOLEY`. Transitions only go toward more general kinds. Pre-filling with `new Array(n).fill(0)` keeps it packed; writing far past the end makes it holey or even dictionary-mode. For numeric-heavy code, **typed arrays** (`Float64Array`, `Int32Array`) give fixed-size, unboxed, contiguous storage.
- **CPython list** is an array of pointers to boxed objects, so a list of a million ints is ~8 MB of pointers plus the int objects. For numeric data use `array.array` or NumPy.
- **Immutable strings are a feature**: they're safe to share across threads, usable as hash keys, and can be interned. The cost is building them; that's what builders and `join` fix.
- **Non-mutating array methods (ES2023)**: `toSorted`, `toReversed`, `toSpliced` and `with(i, v)` return a copy instead of mutating. They're useful in React state updates; they are O(n) copies, so don't use them inside hot loops.
- **In review, ask**: Is there a `shift()` / `pop(0)` / `includes` / `indexOf` inside a loop (hidden O(n²))? Are we copying slices in recursion? Is the string built with `+=` in a loop over user-sized input? Is the array pre-sized when the size is known?
- **In interviews**: say out loud whether you may mutate the input, what the character set is (ASCII lowercase? Unicode?), and whether output space counts toward the space complexity.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 344 | [Reverse String](https://leetcode.com/problems/reverse-string/) | Easy | two pointers swapping inward, in place |
| 242 | [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Easy | 26-slot count array |
| 283 | [Move Zeroes](https://leetcode.com/problems/move-zeroes/) | Easy | read/write pointers |
| 189 | [Rotate Array](https://leetcode.com/problems/rotate-array/) | Medium | triple reverse, O(1) space |
| 238 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Medium | prefix and suffix products ([[Prefix Sum]] idea) |
| 49 | [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Medium | sorted string or count tuple as hash key |
| 151 | [Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string/) | Medium | split, reverse, join; or in-place double reverse |
| 443 | [String Compression](https://leetcode.com/problems/string-compression/) | Medium | read/write pointers, write counts in place |
| 48 | [Rotate Image](https://leetcode.com/problems/rotate-image/) | Medium | transpose then reverse each row |

## Interview questions

> [!question]- Why is appending to a dynamic array O(1) if it sometimes copies everything?
> Because capacity grows geometrically (x1.5 or x2). The copies over `n` appends sum to less than `c * n`, so the **amortized** cost per append is O(1). A single append can still be O(n), which matters for latency-sensitive code.

> [!question]- Why is building a string with `+=` in a loop slow, and what do you do instead?
> Strings are immutable, so each `+=` may allocate a new string and copy the old content: 1 + 2 + ... + n = O(n²). Collect pieces in an array/list and call `join` once (O(n)), or use a `StringBuilder` in Java/C#.

> [!question]- What's the difference between `arr.shift()` and `arr.pop()` in JS?
> `pop` removes the last element in O(1). `shift` removes the first and must re-index every remaining element, O(n). Using `shift` as a queue in BFS turns O(V + E) into O(V² + E). Use a head index or a real deque.

> [!question]- When would you pick a 26-length array over a hash map for counting?
> When the alphabet is small and known (lowercase English). It's faster (no hashing), uses fixed memory, and is O(1) space. Use a map when the character set is large or unknown (Unicode).

> [!question]- How do you rotate an array by k in O(1) extra space?
> Reverse the whole array, then reverse the first `k % n` elements, then the rest. Each element moves exactly to its target.

> [!question]- Why are arrays often faster than linked lists even for workloads with inserts?
> Contiguous memory gives cache locality and prefetching; linked lists chase pointers to random heap addresses, causing cache misses and allocation overhead. For small to medium `n`, shifting a contiguous block with `memmove` is often cheaper than allocating and linking a node.

> [!question]- What does "in-place" mean, and does it make the algorithm O(1) space?
> It means you modify the input rather than allocate a copy. Extra space is O(1) only if you also avoid recursion stacks and auxiliary structures. Also clarify with the interviewer whether mutating the input is allowed.

## Related
- [[DSA Index]]
- [[Big-O & Complexity Analysis]] · [[Linked Lists]] · [[Stacks & Queues]] · [[Hash Tables]]
- Patterns: [[Two Pointers]] · [[Sliding Window]] · [[Prefix Sum]] · [[Binary Search]]
- [[Sorting Algorithms]]
- Frontend: [[Iterators, Generators & Iterator Helpers]] · [[Memory Management & Garbage Collection]] · [[Virtualization & Large Lists]]
- [[LeetCode Crash Course]]

## Sources
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022 (ch. 16, amortized analysis; dynamic tables)
- [Sedgewick & Wayne, Algorithms 4th ed.: Resizing arrays (1.3)](https://algs4.cs.princeton.edu/13stacks/)
- [CPython source: listobject.c (list_resize)](https://github.com/python/cpython/blob/main/Objects/listobject.c)
- [V8 blog: Elements kinds in V8](https://v8.dev/blog/elements-kinds)
- [MDN: Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array) · [MDN: String](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)
- [MDN: Array.prototype.toSorted (ES2023)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/toSorted)
- [Python docs: Sequence types, str](https://docs.python.org/3/library/stdtypes.html#text-sequence-type-str)
- [LeetCode Explore: Interview Crash Course](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/)
- [Tech Interview Handbook: Array cheatsheet](https://www.techinterviewhandbook.org/algorithms/array/)
