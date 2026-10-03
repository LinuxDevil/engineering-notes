---
tags: [dsa, dsa/books]
aliases: [Grokking, Grokking Algorithms Book, Bhargava Algorithms]
updated: 2026-09-27
---
# Grokking Algorithms

> [!tldr] TL;DR
> Book notes on *Grokking Algorithms* by Aditya Y. Bhargava (Manning; 2nd edition February 2024), an illustrated, beginner-friendly introduction to algorithms. It covers binary search and Big-O, arrays vs linked lists, selection sort, recursion and the call stack, quicksort and divide and conquer, hash tables, BFS, trees, Dijkstra, greedy algorithms, dynamic programming and k-nearest neighbours. Each chapter below has my notes (bugs fixed), a short summary where the notes were missing, and a link to the full topic note.

An algorithm is a set of instructions for solving a task.

| Ch | Topic | Full note |
|---|---|---|
| 1 | Binary search, Big-O | [[Binary Search]], [[Big-O & Complexity Analysis]] |
| 2 | Arrays vs linked lists, selection sort | [[Arrays & Strings]], [[Linked Lists]], [[Sorting Algorithms]] |
| 3 | Recursion, the call stack | [[Recursion & Backtracking]] |
| 4 | Quicksort, divide and conquer | [[Sorting Algorithms]] |
| 5 | Hash tables | [[Hash Tables]] |
| 6 | Breadth-first search | [[Graphs]], [[Graph BFS-DFS patterns]] |
| 7 | Trees (2nd ed.) | [[Trees & Binary Search Trees]], [[Tree Traversal - DFS & BFS]] |
| 8 | Balanced trees (2nd ed.) | [[Trees & Binary Search Trees]] |
| 9 | Dijkstra's algorithm | [[Graphs]] |
| 10 | Greedy algorithms, NP-complete problems | [[Greedy]] |
| 11 | Dynamic programming | [[Dynamic Programming]] |
| 12 | k-nearest neighbours | (machine learning intro) |
| 13 | Where to go next | |

## Chapter 1: Introduction, binary search, Big-O

### Binary search

Its input is a **sorted** list. It searches for the position of an element and returns it; if not found it returns `null`.

It works by halving the search area each step. Look at the middle element of `[0,1,2,3,4,5,6,7,8,9]`: if it's the target, done; if it's bigger than the target, the target can only be in the left half; if smaller, only in the right half. Keep halving until you find it or the range is empty.

- Simple (linear) search: up to `n` guesses.
- Binary search: up to `log₂ n` guesses. For 4 billion items, that's 32 guesses.

### Logarithms and exponentials

**Exponentials**: how many 10s do we multiply together to get 100? The answer is 2: 10 × 10 = 10² = 100.

**Logarithms** are the flip of exponentials: log₁₀ 100 = 2. In this book (and in Big-O generally) "log" means log₂: log₂ 8 = 3 because 2³ = 8.

![[Screenshot_2025-02-06_at_2.58.15_PM.png]]

### Code

> [!warning] Fixed bug
> The original had `const mid = low + high;`, which isn't the middle and jumps out of range. The middle is `Math.floor((low + high) / 2)`.

```javascript
function binarySearch(list, item) {
  let low = 0;
  let high = list.length - 1;

  while (low <= high) {
    const mid = Math.floor((low + high) / 2);
    const guess = list[mid];

    if (guess === item) return mid;
    if (guess > item) high = mid - 1;   // too high: search the left half
    else low = mid + 1;                 // too low: search the right half
  }
  return null;
}

console.log(binarySearch([0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 5)); // 5
```

### Big-O (the rest of chapter 1)
- Big-O tells you **how fast an algorithm grows** (number of operations), not the speed in seconds, and it describes the **worst case**.
- Common run times, fastest to slowest: O(log n), O(n), O(n log n), O(n²), O(n!).
- The traveling salesperson brute force is O(n!): there's no known fast exact algorithm.

More in [[Big-O & Complexity Analysis]] and [[Binary Search]].

## Chapter 2: Selection sort (arrays vs linked lists)

Arrays take a fixed, **contiguous** spot in memory. If you want to grow one and the next slots are taken, you have to move it to another address where the full length fits (dynamic arrays do this for you, see [[Arrays & Strings]]).

In a linked list, items can be anywhere in memory: each item stores the address of the next one.

> Linked lists are great if you're going to read all the items one at a time: you can read one item, follow the address to the next item, and so on. But if you're going to keep jumping around, linked lists are terrible

There are two different types of access: random access and sequential access.

- **Sequential access** means reading the elements one by one, starting at the first element.
	- Linked lists can only do sequential access.
- **Random access** means you can jump directly to the 10th element.
	- Arrays are the champion here.

| | Array | Linked list |
|---|---|---|
| Reading | O(1) | O(n) |
| Insertion | O(n) | O(1) (if you have the node) |
| Deletion | O(n) | O(1) (if you have the node) |

The book's hybrid example: an array of linked lists (Facebook usernames bucketed by first letter), which is basically how a hash table with chaining works.

### Selection sort

Selection sort is a neat algorithm, but it's not very fast: O(n²). Quicksort is a faster sorting algorithm that takes O(n log n) on average (next chapters).

The idea: repeatedly find the smallest remaining item and move it to the sorted part. For example in a music player, you have a list of songs and their play counts and want them sorted from most to least played: go through the whole list to find the most-played song, add it to a new list, repeat.

![[Screenshot_2025-02-16_at_9.03.55_AM.png]]

So first define a function to find the smallest item's index:

```javascript
function findSmallest(list) {
  let smallest = list[0];
  let smallestIndex = 0;
  for (let i = 1; i < list.length; i++) {
    if (list[i] < smallest) {
      smallest = list[i];
      smallestIndex = i;
    }
  }
  return smallestIndex;
}
```

Then write selection sort with it:

> [!warning] Fixed bug
> The original pushed the **index** returned by `findSmallest` into the new array and never removed the item from the list, so it returned the same index `n` times. The fix: remove the smallest item from a copy and push its **value**.

```javascript
function selectionSort(list) {
  const copy = [...list];            // don't destroy the caller's array
  const sorted = [];
  while (copy.length > 0) {
    const i = findSmallest(copy);
    sorted.push(copy.splice(i, 1)[0]); // remove it and push the value
  }
  return sorted;
}

console.log(selectionSort([5, 3, 6, 2, 10])); // [2, 3, 5, 6, 10]
```

Or do it **in place** (O(1) extra space), which is the usual version:

```javascript
function selectionSortInPlace(list) {
  for (let i = 0; i < list.length; i++) {
    let smallestIndex = i;
    for (let j = i + 1; j < list.length; j++) {
      if (list[j] < list[smallestIndex]) smallestIndex = j;
    }
    // in-place swap
    [list[i], list[smallestIndex]] = [list[smallestIndex], list[i]];
  }
  return list;
}
```

Why O(n²): you scan n items, then n−1, then n−2... which sums to about n²/2 and constants are dropped.

## Chapter 3: Recursion

A function calling itself until the **base case** is reached.

> Loops may achieve a performance gain for your program. Recursion may achieve a performance gain for your programmer. Choose which is more important in your situation!

Every recursive function has two parts: the base case and the recursive case.

- **Recursive case**: the function calls itself (on a smaller problem).
- **Base case**: the function doesn't call itself, it stops. Without it you recurse forever (stack overflow).

![[Screenshot_2025-02-16_at_9.40.35_AM.png]]

### The stack

The stack is like a pile of sticky notes: when you insert (**push**) an item it goes on top; when you read an item you only read the topmost one, and it's taken off (**pop**). Last in, first out. See [[Stacks & Queues]].

![[Screenshot_2025-02-16_at_9.44.48_AM.png]]

### The call stack

> The computer internally uses a stack that is called the "call stack" to keep track of the functions run and their allocated memory address to them.

```python
def greet(name):
    print("hello, " + name + "!")
    greet2(name)
    print("getting ready to say bye...")
    bye()

def greet2(name):
    print("how are you, " + name + "?")

def bye():
    print("ok bye!")
```

Whenever you call a function, the computer allocates a box of memory (a **stack frame**) for it, holding its arguments and local variables.

![[Screenshot_2025-02-16_at_9.47.35_AM.png]]

Every time you call a function, its frame gets pushed onto the stack of functions.

![[Screenshot_2025-02-16_at_9.48.03_AM.png]]

Why does `greet2` sit on top of `greet`? Because `greet` called it internally:

> When you called the greet2 function, the greet function was partially completed. This is the big idea behind this section: when you call a function from another function, the calling function is paused in a partially completed state.

![[Screenshot_2025-02-16_at_9.49.49_AM.png]]

Recursion uses the call stack too, for example the factorial function:

```javascript
function factorial(n) {
  if (n <= 1) return 1;            // base case
  return n * factorial(n - 1);     // recursive case
}
```

(The original used `factorial(--number)`, which works but mutates the variable before the multiplication reads it; `n - 1` is clearer.)

![[Screenshot_2025-02-16_at_9.51.46_AM.png]]

> [!tip] 💡
> Using the stack is convenient, but there's a cost: saving all that info can take up a lot of memory. Each call is a frame; very deep recursion can overflow the stack. Rewrite with a loop, or use tail recursion where the language optimizes it.

### Tail recursion

> [!tip] 💡
> **Tail recursion** is defined as a recursive function in which the recursive call is the last statement that is executed by the function. So basically nothing is left to execute after the recursion call.

Tail-recursive functions can be better than non-tail-recursive ones because the compiler can optimize them (**tail call elimination**).

> [!tip] 💡
> Compilers usually execute recursive procedures by using a **stack**. This stack consists of all the pertinent information, including the parameter values, for each recursive call. When a procedure is called, its information is **pushed** onto a stack, and when the function terminates the information is **popped** out of the stack. Thus for non-tail-recursive functions, the **stack depth** (maximum amount of stack space used at any time during execution) is larger.

> The idea used by compilers to optimize tail-recursive functions is simple, since the recursive call is the last statement, there is nothing left to do in the current function, so saving the current function's stack frame is of no use ([See this for more details](https://www.geeksforgeeks.org/tail-call-elimination/)).

So a non-tail-recursive factorial is:

```javascript
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}
```

> [!tip] 💡
> Although it looks tail recursive at first look, if we take a closer look we can see that the value returned by `factorial(n - 1)` is used in `factorial(n)` (it still has to multiply). So the call to `factorial(n - 1)` is not the last thing done by `factorial(n)`.

To write it as tail recursion, carry the result in an **accumulator**:

> [!warning] Fixed bugs
> The original returned an undefined variable `a`, the wrapper always called `factTailRec(5, 1)` ignoring its argument, and `console.log(fact(5);` was missing a parenthesis.

```javascript
function factTailRec(n, acc) {
  if (n <= 1) return acc;                // accumulated result
  return factTailRec(n - 1, n * acc);    // the call is the very last thing
}

// wrapper with the initial accumulator
function fact(n) {
  return factTailRec(n, 1);
}

console.log(fact(5)); // 120
```

> [!tip] 💡
> The idea is to use one more argument and accumulate the factorial value in it. When `n` reaches the base case, return the accumulated value.

> [!warning] Reality check
> ES2015 specifies proper tail calls in strict mode, but **V8 (Chrome, Node.js) and SpiderMonkey (Firefox) don't implement them**; only Safari's JavaScriptCore does. **CPython** never does tail-call elimination either (default recursion limit 1000). So in JS and Python, tail recursion does not save stack space; convert deep recursion to a loop. See [[Recursion & Backtracking]].

## Chapter 4: Quicksort and divide and conquer

Divide and conquer (D&C) algorithms are recursive. Two steps:

- Figure out the **base case**. This should be the simplest possible case.
- **Divide** (reduce) your problem until it becomes the base case.

> With every recursive call, you have to reduce your problem

The book's warm-up examples: splitting a farm plot into the largest even square plots (Euclid's algorithm), and recursive `sum`, `count` and `max` of a list.

Quicksort uses D&C: split the problem into partitions around a **pivot**. The base case is an empty array or an array with one item, which doesn't need sorting.

Take the pivot, then find the elements smaller than the pivot and the elements larger than it:

![[Screenshot_2025-02-16_at_1.53.21_PM.png]]

- a sub-array of all the numbers less than the pivot
- the pivot
- a sub-array of all the numbers greater than (or equal to) the pivot

Sort each sub-array recursively, then combine: `smaller + pivot + bigger`.

So the steps are:

1. Pick a pivot.
2. Partition the array into two sub-arrays.
3. Call quicksort recursively on the two sub-arrays.

```javascript
function quickSort(array) {
  if (array.length < 2) return array;  // base case

  const pivot = array[0];
  const less = [];
  const greater = [];

  for (let i = 1; i < array.length; i++) {
    if (array[i] < pivot) less.push(array[i]);
    else greater.push(array[i]);
  }

  return [...quickSort(less), pivot, ...quickSort(greater)];
}

console.log(quickSort([10, 5, 2, 3])); // [2, 3, 5, 10]
```

**Choosing the pivot matters.** Picking the first element on an already-sorted array gives one empty side every time: O(n) levels × O(n) work = **O(n²)** worst case. Picking a **random** pivot gives O(log n) levels on average → **O(n log n)** average. This version also uses O(n) extra memory; production quicksorts partition in place (see [[Sorting Algorithms]]).

### Inductive proofs

**Inductive proofs** are a way to prove your algorithm works. They have two steps: the **base case** and the **inductive case**. Climbing a ladder: *if my legs are on a rung, I can put them on the next rung* (inductive case), and *my legs are on rung 1* (base case). Together they prove I can climb the whole ladder, one rung at a time. For quicksort: it works on arrays of size 0 and 1, and if it works on smaller arrays it works on the partitions, so it works on any size.

> [!tip] 💡
> Quicksort versus merge sort is one example where constants matter. Quicksort has a smaller constant than merge sort, so if they're both O(n log n), quicksort is faster in practice. And it hits the average case way more often than the worst case.

## Chapter 5: Hash tables

A hash table is an array plus a **hash function** that turns a key into an array index: a key-value "Map".

A **hash function** is a function that maps strings (or any key) to numbers. Requirements:

- **Consistent**: the same input must always yield the same output.
- It should map **different** inputs to **different** numbers as much as possible (spread keys evenly).
- It should only return numbers within the array's size.

![[Screenshot_2025-02-19_at_10.52.54_AM.png]]

Use cases from the book (my summary):
- **Lookups**: a phone book, DNS (domain name → IP address).
- **Preventing duplicates**: checking whether someone has already voted.
- **Caching**: remember the result of an expensive computation or web page render and return it on the next request.

**Collisions**: two keys hash to the same slot. The simplest fix is to store a linked list in that slot (chaining). If lists get long, lookups degrade to O(n), so a good hash function and a low **load factor** matter.

**Load factor** = items / slots. When it goes above about 0.7, **resize**: allocate an array about twice as large and re-insert everything. Resizing is expensive but rare, so inserts stay O(1) on average.

| | Average | Worst |
|---|---|---|
| Search | O(1) | O(n) |
| Insert | O(1) | O(n) |
| Delete | O(1) | O(n) |

In practice you use the language's built-in: `Map`/`Set` in JavaScript, `dict`/`set` in Python. More in [[Hash Tables]].

## Chapter 6: Breadth-first search (summary)
- A **graph** models connections: nodes and edges. Directed edges have a direction; undirected ones don't.
- **BFS** answers two questions: *is there a path from A to B?* and *what's the shortest path (fewest edges)?*
- It explores neighbours first, then neighbours of neighbours, using a **queue** (FIFO). Mark nodes as visited or you can loop forever on cycles.
- The book's example: find a mango seller in your network, checking friends before friends of friends.
- Running time **O(V + E)**.
- **Topological sort** orders tasks so each comes after its dependencies (a DAG, like getting dressed).

More in [[Graphs]] and [[Graph BFS-DFS patterns]].

## Chapter 7: Trees (2nd ed., summary)
- A tree is a graph with no cycles where each node (except the root) has exactly one parent. File systems are trees.
- **Depth-first search** explores one branch fully before backtracking; it's the natural fit for walking a directory tree recursively.
- **Binary trees** have at most two children; **Huffman coding** uses a binary tree to give frequent characters shorter codes.

More in [[Trees & Binary Search Trees]] and [[Tree Traversal - DFS & BFS]].

## Chapter 8: Balanced trees (2nd ed., summary)
- A **binary search tree** keeps smaller keys on the left and larger on the right, giving O(log n) search and insert, **if it stays balanced**. Inserting sorted data into a plain BST produces a linked list: O(n).
- **Balanced trees** (AVL trees rebalance with rotations) guarantee O(log n) height. **B-trees** are wide, shallow balanced trees used by databases, to minimise disk reads.

## Chapter 9: Dijkstra's algorithm (summary)
- Finds the **shortest path in a weighted graph** (fewest total weight, not fewest edges).
- Repeatedly take the cheapest unprocessed node, update its neighbours' costs if going through it is cheaper, mark it processed. Track **parents** to reconstruct the path.
- Works only with **non-negative weights**; with negative weights use **Bellman-Ford**.

More in [[Graphs]].

## Chapter 10: Greedy algorithms (summary)
- At each step, pick the locally optimal choice. Examples: classroom scheduling (pick the class that ends soonest), and set covering (radio stations).
- Greedy is simple and fast, but only **approximately** right for some problems.
- **NP-complete** problems (set cover, traveling salesperson) have no known fast exact solution; use approximation algorithms. Signs: "all combinations", "every possible order", can't break into smaller subproblems.

More in [[Greedy]].

## Chapter 11: Dynamic programming (summary)
- Solve a hard problem by solving subproblems first and storing their answers in a **grid** (table). The book's example: the knapsack problem (maximise the value of items that fit in a bag), plus longest common substring/subsequence.
- Every DP solution involves a grid; the values in the cells are usually what you're optimising; each cell is a subproblem.
- Works only when subproblems are **independent** (discrete items, not fractions).

More in [[Dynamic Programming]].

## Chapter 12: k-nearest neighbours (summary)
- Classify or predict something by looking at the `k` most similar data points (features as coordinates, distance via Pythagoras or cosine similarity).
- Used for recommendations (Netflix-style) and regression. Picking good **features** is the key; a light intro to machine learning.

## Related
- [[DSA Index]] · [[LeetCode Crash Course]]
- [[Binary Search]] · [[Sorting Algorithms]] · [[Recursion & Backtracking]] · [[Hash Tables]] · [[Graphs]] · [[Dynamic Programming]] · [[Greedy]]
- [[Execution Context & Call Stack]] (the call stack in JavaScript)

## Sources
- Aditya Y. Bhargava, *Grokking Algorithms*, 2nd ed., Manning, February 2024: [manning.com](https://www.manning.com/books/grokking-algorithms-second-edition)
- [Official code repository (egonschiele/grokking_algorithms)](https://github.com/egonschiele/grokking_algorithms)
- [GeeksforGeeks: Tail call elimination](https://www.geeksforgeeks.org/tail-call-elimination/)
- [Compatibility table: ES6 proper tail calls](https://compat-table.github.io/compat-table/es6/) (only Safari supports them)
- [Python docs: sys.setrecursionlimit](https://docs.python.org/3/library/sys.html#sys.setrecursionlimit)
