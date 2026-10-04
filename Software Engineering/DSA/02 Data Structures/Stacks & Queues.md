---
title: "Stacks & Queues"
tags: [dsa, dsa/data-structures]
aliases: [Stack, Queue, Deque, Stacks, Queues, LIFO, FIFO, Circular Buffer, Ring Buffer]
updated: 2026-09-27
---
# Stacks & Queues

<Note>
**TL;DR**

A **stack** is LIFO (last in, first out): push and pop at the same end. A **queue** is FIFO (first in, first out): add at the back, remove from the front. A **deque** allows both ends. All core operations are **O(1)** when implemented right. In JS, an array is a fine stack, but `array.shift()` is **O(n)**, so a queue built on `shift` silently turns BFS quadratic; use a head index, a circular buffer, or a linked list. In Python, use `list` for stacks and `collections.deque` for queues.
</Note>

## The idea

- **Stack**: a pile of plates. You add to the top and take from the top. Undo history, the browser back button, and the function call stack ([Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack)) all work this way.
- **Queue**: a line at a shop. First person in is first served. Print queues, task schedulers, message queues and BFS all work this way.
- **Deque** (double-ended queue, "deck"): a line where people can join or leave at either end. Sliding-window maximum uses it ([Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue)).

```text
Stack (LIFO)            Queue (FIFO)
   push  pop              enqueue                 dequeue
     |   ^                   |                       ^
     v   |                   v                       |
   +-------+              +---+---+---+---+---+
   |   C   |  <- top      | E | D | C | B | A |  -> front
   +-------+              +---+---+---+---+---+
   |   B   |              back
   +-------+
   |   A   |
   +-------+
```

## How it works

### Stack operations

| Operation | JS array | Python list |
|---|---|---|
| push | `arr.push(x)` | `lst.append(x)` |
| pop | `arr.pop()` | `lst.pop()` |
| peek | `arr[arr.length - 1]` or `arr.at(-1)` | `lst[-1]` |
| empty? | `arr.length === 0` | `not lst` |

All O(1) (amortized for push). The end of a dynamic array is the natural top ([Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings)).

### Queue operations and the JS `shift` pitfall

| Operation | Correct choice (Python) | JS naive | JS correct |
|---|---|---|---|
| enqueue | `dq.append(x)` O(1) | `arr.push(x)` O(1) | same |
| dequeue | `dq.popleft()` O(1) | `arr.shift()` **O(n)** | head index, ring buffer, or linked list |
| peek front | `dq[0]` | `arr[0]` | `arr[head]` |

`shift()` removes index 0 and conceptually re-indexes every other element. V8 has some tricks for small arrays (it can move the start of the backing store "left-trimming"), but you cannot rely on them, and on large arrays it is linear. Python `list.pop(0)` is also O(n). In a BFS over 10^5 nodes that is up to 10^10 operations.

### Circular buffer (ring buffer)

A fixed-capacity array plus `head` and `size`. Indexes wrap with modulo, so no element ever moves.

```text
capacity 5, head=3, size=3
index:   0     1     2     3     4
       +-----+-----+-----+-----+-----+
       |  E  |     |     |  C  |  D  |
       +-----+-----+-----+-----+-----+
                           ^head
order: C (3), D (4), E (0)      tail index = (head + size) % cap = 1
```

Grow it by allocating double the capacity and copying items in order (`head` becomes 0). That keeps enqueue amortized O(1), like a dynamic array.

### Queue via two stacks

Keep an `inStack` for pushes and an `outStack` for pops. When `outStack` is empty, pour everything from `inStack` into it (which reverses the order, so the oldest item ends on top). Each element is moved at most once, so dequeue is **amortized O(1)**.

{/* diagram source: /diagrams/diagram-190.mmd */}
![diagram](/diagrams/diagram-190.svg)

## Complexity

| Structure | push / enqueue | pop / dequeue | peek | search | space |
|---|---|---|---|---|---|
| Stack (dynamic array) | O(1) amortized | O(1) | O(1) | O(n) | O(n) |
| Queue with `shift()` (JS) / `pop(0)` (Python) | O(1) | **O(n)** | O(1) | O(n) | O(n) |
| Queue (ring buffer / linked list / deque) | O(1) amortized | O(1) | O(1) | O(n) | O(n) |
| Queue via two stacks | O(1) | O(1) amortized, O(n) worst | O(1) amortized | O(n) | O(n) |
| Deque (Python `deque`) | O(1) both ends | O(1) both ends | O(1) | O(n); index `dq[i]` is O(n) toward the middle | O(n) |

## Implementation

### Stack

```ts
class Stack<T> {
  private items: T[] = [];
  push(x: T): void { this.items.push(x); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  get size(): number { return this.items.length; }
  isEmpty(): boolean { return this.items.length === 0; }
}
```

```python
# A plain list is the idiomatic stack in Python.
stack: list[int] = []
stack.append(1)
stack.append(2)
top = stack[-1]      # peek -> 2
stack.pop()          # -> 2
is_empty = not stack
```

### Valid Parentheses (LeetCode 20): the classic stack problem

```ts
function isValid(s: string): boolean {
  const pairs: Record<string, string> = { ")": "(", "]": "[", "}": "{" };
  const stack: string[] = [];
  for (const ch of s) {
    if (ch in pairs) {
      if (stack.pop() !== pairs[ch]) return false;
    } else {
      stack.push(ch);
    }
  }
  return stack.length === 0;
}
```

```python
def is_valid(s: str) -> bool:
    pairs = {")": "(", "]": "[", "}": "{"}
    stack = []
    for ch in s:
        if ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
        else:
            stack.append(ch)
    return not stack
```

### Min Stack (LeetCode 155): O(1) getMin

Store, next to each value, the minimum at that depth.

```ts
class MinStack {
  private stack: [value: number, min: number][] = [];
  push(val: number): void {
    const min = this.stack.length ? Math.min(val, this.getMin()) : val;
    this.stack.push([val, min]);
  }
  pop(): void { this.stack.pop(); }
  top(): number { return this.stack[this.stack.length - 1][0]; }
  getMin(): number { return this.stack[this.stack.length - 1][1]; }
}
```

```python
class MinStack:
    def __init__(self):
        self.stack: list[tuple[int, int]] = []

    def push(self, val: int) -> None:
        current_min = min(val, self.stack[-1][1]) if self.stack else val
        self.stack.append((val, current_min))

    def pop(self) -> None:
        self.stack.pop()

    def top(self) -> int:
        return self.stack[-1][0]

    def getMin(self) -> int:
        return self.stack[-1][1]
```

### Queue in JS: the cheap fix (head index)

For BFS you often don't need a full class. Keep a read pointer instead of shifting.

```ts
function bfsOrder(start: number, adj: number[][]): number[] {
  const queue: number[] = [start];
  const seen = new Set<number>([start]);
  let head = 0;                         // instead of queue.shift()
  while (head < queue.length) {
    const node = queue[head++];
    for (const next of adj[node]) {
      if (!seen.has(next)) { seen.add(next); queue.push(next); }
    }
  }
  return queue;                         // the visit order
}
```

```python
from collections import deque

def bfs_order(start: int, adj: list[list[int]]) -> list[int]:
    queue = deque([start])
    seen = {start}
    order = []
    while queue:
        node = queue.popleft()          # O(1), unlike list.pop(0)
        order.append(node)
        for nxt in adj[node]:
            if nxt not in seen:
                seen.add(nxt)
                queue.append(nxt)
    return order
```

The head-index trick never frees the consumed prefix. That's fine for one BFS; for a long-lived queue, use a ring buffer.

### Circular buffer deque (growable)

JS has no built-in deque. This one supports both ends in O(1) amortized.

```ts
class Deque<T> {
  private buf: (T | undefined)[];
  private head = 0;
  private count = 0;

  constructor(capacity = 8) {
    this.buf = new Array(capacity);
  }

  get size(): number { return this.count; }

  private grow(): void {
    const next = new Array<T | undefined>(this.buf.length * 2);
    for (let i = 0; i < this.count; i++) next[i] = this.buf[(this.head + i) % this.buf.length];
    this.buf = next;
    this.head = 0;
  }

  pushBack(x: T): void {
    if (this.count === this.buf.length) this.grow();
    this.buf[(this.head + this.count) % this.buf.length] = x;
    this.count++;
  }

  pushFront(x: T): void {
    if (this.count === this.buf.length) this.grow();
    this.head = (this.head - 1 + this.buf.length) % this.buf.length;
    this.buf[this.head] = x;
    this.count++;
  }

  popFront(): T | undefined {
    if (this.count === 0) return undefined;
    const x = this.buf[this.head];
    this.buf[this.head] = undefined;              // release reference
    this.head = (this.head + 1) % this.buf.length;
    this.count--;
    return x;
  }

  popBack(): T | undefined {
    if (this.count === 0) return undefined;
    const i = (this.head + this.count - 1) % this.buf.length;
    const x = this.buf[i];
    this.buf[i] = undefined;
    this.count--;
    return x;
  }

  peekFront(): T | undefined { return this.count ? this.buf[this.head] : undefined; }
  peekBack(): T | undefined {
    return this.count ? this.buf[(this.head + this.count - 1) % this.buf.length] : undefined;
  }
}
```

```python
# In Python, use collections.deque. For a fixed-size ring buffer, pass maxlen:
from collections import deque

dq = deque()
dq.append(1)         # push back
dq.appendleft(0)     # push front
dq.pop()             # pop back  -> 1
dq.popleft()         # pop front -> 0

last_3 = deque(maxlen=3)        # ring buffer: oldest item drops automatically
for x in range(5):
    last_3.append(x)
print(list(last_3))             # [2, 3, 4]


class CircularQueue:
    """Fixed-capacity ring buffer (LeetCode 622 style)."""

    def __init__(self, k: int):
        self.buf = [0] * k
        self.head = 0
        self.count = 0

    def enqueue(self, value: int) -> bool:
        if self.count == len(self.buf):
            return False
        self.buf[(self.head + self.count) % len(self.buf)] = value
        self.count += 1
        return True

    def dequeue(self) -> bool:
        if self.count == 0:
            return False
        self.head = (self.head + 1) % len(self.buf)
        self.count -= 1
        return True

    def front(self) -> int:
        return self.buf[self.head] if self.count else -1

    def rear(self) -> int:
        return self.buf[(self.head + self.count - 1) % len(self.buf)] if self.count else -1
```

### Queue via two stacks (LeetCode 232)

```ts
class MyQueue {
  private inStack: number[] = [];
  private outStack: number[] = [];

  push(x: number): void { this.inStack.push(x); }

  private shiftStacks(): void {
    if (this.outStack.length === 0) {
      while (this.inStack.length) this.outStack.push(this.inStack.pop()!);
    }
  }

  pop(): number { this.shiftStacks(); return this.outStack.pop()!; }
  peek(): number { this.shiftStacks(); return this.outStack[this.outStack.length - 1]; }
  empty(): boolean { return this.inStack.length === 0 && this.outStack.length === 0; }
}
```

```python
class MyQueue:
    def __init__(self):
        self.in_stack: list[int] = []
        self.out_stack: list[int] = []

    def push(self, x: int) -> None:
        self.in_stack.append(x)

    def _shift(self) -> None:
        if not self.out_stack:
            while self.in_stack:
                self.out_stack.append(self.in_stack.pop())

    def pop(self) -> int:
        self._shift()
        return self.out_stack.pop()

    def peek(self) -> int:
        self._shift()
        return self.out_stack[-1]

    def empty(self) -> bool:
        return not self.in_stack and not self.out_stack
```

Only pour when `outStack` is empty. Pouring every time breaks the order and the amortized bound.

## Trade-offs & pitfalls

- **`shift()` / `pop(0)` in a loop**: the most common hidden O(n²) in JS and Python solutions.
- **Popping an empty stack**: JS returns `undefined` (silently), Python raises `IndexError`. Check emptiness first, especially in parentheses matching.
- **Ring buffer full vs empty**: with only `head` and `tail` indexes, `head === tail` is ambiguous. Keep a `count` (as above) or leave one slot unused.
- **Negative modulo in JS**: `-1 % 5 === -1`. Use `(i - 1 + cap) % cap`. Python's `%` is already non-negative for a positive divisor.
- **Recursion is a stack**: deep recursion can overflow (`RangeError: Maximum call stack size exceeded` in JS, `RecursionError` in Python at ~1000 frames). Convert to an explicit stack for deep DFS.
- **Python `queue.Queue`** is a thread-safe queue with locks, slower than `deque` for single-threaded algorithms. `asyncio.Queue` is for coroutines. Use `deque` in interviews.
- **Priority queue is not a queue**: "queue" by priority is a heap ([Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues)).

## Senior perspective

- **Stacks everywhere**: the call stack, expression parsing (shunting-yard, RPN), undo/redo (two stacks), DFS, backtracking, matching tags in HTML parsers, and the browser history API.
- **Queues everywhere**: the event loop's task and microtask queues, message brokers (Kafka, SQS, RabbitMQ), job queues (BullMQ, Celery), network packet buffers, rate limiters and BFS.
- **Ring buffers are the systems workhorse**: fixed memory, no allocation after startup, cache friendly. Used in audio/video pipelines, logging (keep last N entries), Linux `io_uring` submission/completion queues, the LMAX Disruptor, and metrics windows. Python's `deque(maxlen=N)` is a quick ring buffer.
- **Bounded vs unbounded**: an unbounded queue in production is a memory leak waiting for a traffic spike. A senior asks: what is the capacity, and what happens when it's full? Options are block (backpressure), drop newest, drop oldest, or reject with an error (HTTP 429/503).
- **Concurrency**: single-producer/single-consumer ring buffers can be lock-free with just two atomic indexes. Multi-producer queues need CAS loops or locks.
- **CPython `deque`** is a doubly linked list of 64-slot blocks: O(1) at both ends and much better locality than one node per item, but `dq[i]` in the middle is O(n).
- **In interviews**: whenever you write BFS in JS, say "I'll use a head index instead of `shift` because `shift` is O(n)". It signals you know the runtime, not just the algorithm.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 20 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Easy | push openers, match on closers |
| 232 | [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) | Easy | in/out stacks, amortized O(1) |
| 225 | [Implement Stack using Queues](https://leetcode.com/problems/implement-stack-using-queues/) | Easy | rotate queue after each push |
| 155 | [Min Stack](https://leetcode.com/problems/min-stack/) | Medium | store running min with each value |
| 150 | [Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/) | Medium | operand stack; truncate division toward zero |
| 622 | [Design Circular Queue](https://leetcode.com/problems/design-circular-queue/) | Medium | ring buffer with head and count |
| 394 | [Decode String](https://leetcode.com/problems/decode-string/) | Medium | stack of (count, partial string) |
| 933 | [Number of Recent Calls](https://leetcode.com/problems/number-of-recent-calls/) | Easy | queue, drop timestamps older than 3000 ms |

## Interview questions

<Accordion title="Why is using `array.shift()` as a queue a problem in JS?">
`shift` removes index 0 and must move every other element down one slot, O(n). Used as a BFS queue it makes the traversal O(n²). Use a head index, a ring buffer, or a linked-list queue.
</Accordion>

<Accordion title="Implement a queue with two stacks. What's the complexity?">
Push to `in`. On pop/peek, if `out` is empty, move everything from `in` to `out` (reversing order), then pop from `out`. Each element moves at most once, so operations are amortized O(1); a single pop can be O(n).
</Accordion>

<Accordion title="What is a circular buffer and when would you use it?">
A fixed-size array with wrapping head/tail indexes. Enqueue and dequeue are O(1) with no shifting and no allocation. Used for streaming data, bounded logs, audio buffers, producer/consumer pipelines and "last N items" windows.
</Accordion>

<Accordion title="How do you get O(1) `getMin` on a stack?">
Store each value together with the minimum of the stack at that point (or keep a second stack of minimums). The min is always on top.
</Accordion>

<Accordion title="Stack vs queue: which gives DFS and which gives BFS?">
A stack (explicit or the call stack) gives DFS: go deep first. A queue gives BFS: visit level by level, which finds shortest paths in unweighted graphs.
</Accordion>

<Accordion title="What should a production queue do when it's full?">
Decide a policy explicitly: block producers (backpressure), drop newest, drop oldest, or reject with an error. Unbounded queues hide overload until memory runs out.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis) · [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings) · [Linked Lists](/Software%20Engineering/DSA/02%20Data%20Structures/Linked%20Lists) · [Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues) · [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs)
- Patterns: [Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) · [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns) · [Tree Traversal - DFS & BFS](/Software%20Engineering/DSA/03%20Algorithms/Tree%20Traversal%20-%20DFS%20%26%20BFS) · [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window)
- Frontend: [Execution Context & Call Stack](/Software%20Engineering/Frontend/01%20JavaScript/Execution%20Context%20%26%20Call%20Stack)
- [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course)

## Sources
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022 (ch. 10, stacks and queues)
- [Sedgewick & Wayne, Algorithms 4th ed.: Bags, Queues and Stacks](https://algs4.cs.princeton.edu/13stacks/)
- [Python docs: collections.deque](https://docs.python.org/3/library/collections.html#collections.deque)
- [Python docs: Using lists as queues](https://docs.python.org/3/tutorial/datastructures.html#using-lists-as-queues)
- [MDN: Array.prototype.shift](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/shift)
- [LMAX Disruptor technical paper](https://lmax-exchange.github.io/disruptor/disruptor.html)
- [LeetCode Explore: Queue & Stack card](https://leetcode.com/explore/learn/card/queue-stack/)
- [Tech Interview Handbook: Stack](https://www.techinterviewhandbook.org/algorithms/stack/) · [Queue](https://www.techinterviewhandbook.org/algorithms/queue/)
