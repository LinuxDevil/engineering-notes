---
title: "Greedy"
tags: [dsa, dsa/patterns]
aliases: [Greedy algorithms, Greedy pattern, Greedy choice property, Exchange argument, Interval scheduling]
updated: 2026-09-27
---
# Greedy

<Note>
**TL;DR**

A greedy algorithm makes the **locally best choice at each step and never reconsiders it**. It is correct only when the problem has the **greedy choice property** (some optimal solution starts with the greedy choice) and **optimal substructure**; you usually justify it with an **exchange argument**. Greedy solutions are short and fast (often O(n) or O(n log n) with a sort), but they fail silently on problems that need [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming), such as coin change with arbitrary coins.
</Note>

## The idea

Greedy = "take the best-looking option now". Examples:

- Paying change with the **largest coin first**.
- Picking the meeting that **ends earliest** to fit the most meetings in a day.
- Walking through an array and tracking the **farthest reachable index** (jump game).

The hard part is not coding; it is knowing **when the local choice is safe**.

{/* diagram source: /diagrams/diagram-202.mmd */}
![diagram](/diagrams/diagram-202.svg)

### When greedy fails: coin change

Coins `[1, 3, 4]`, amount `6`.

```text
greedy (largest first):  4 + 1 + 1  = 3 coins
optimal:                 3 + 3      = 2 coins
```

Greedy commits to `4` and cannot undo it. [322. Coin Change](https://leetcode.com/problems/coin-change/) therefore needs DP. Greedy does work for **canonical** coin systems like US or euro coins, which is why cashiers get away with it.

## How it works

### 1. Greedy choice property and optimal substructure
- **Greedy choice property**: there is an optimal solution that contains the greedy first choice.
- **Optimal substructure**: after making that choice, the rest is the same problem on a smaller input, and an optimal solution to the whole contains an optimal solution to the rest.

If both hold, induction gives you correctness: greedy choice, then solve the remainder the same way.

### 2. Exchange argument (proof sketch)
Take any optimal solution OPT that differs from the greedy solution G. Find the first place they differ and **swap** OPT's choice for G's choice. Show the result is still valid and **no worse**. Repeating the swap turns OPT into G without losing quality, so G is optimal.

**Example: interval scheduling (maximum non-overlapping meetings).** Greedy: sort by end time, take a meeting if it starts after the last taken one ends.

- Let `g1` be the meeting that ends first overall. Let OPT's first meeting be `o1`.
- `g1` ends no later than `o1` (it ends first of all meetings).
- Replace `o1` with `g1` in OPT. `g1` ends earlier, so it cannot conflict with OPT's second meeting. OPT' is valid and has the same size.
- Now both solutions start with `g1`; remove it and all meetings overlapping it, and repeat the argument on the rest. So greedy picks as many meetings as OPT.

### 3. Common greedy shapes
| Shape | Greedy rule | Example |
|---|---|---|
| Reachability | track farthest index reachable so far | Jump Game |
| Minimum jumps / BFS levels | extend the current window to the farthest reach | Jump Game II |
| Running balance | reset start when the balance goes negative | Gas Station, Kadane's max subarray |
| Interval selection | sort by end, keep if no conflict | Non-overlapping Intervals, arrows |
| Most frequent first | schedule the most frequent item, fill gaps | Task Scheduler, Reorganize String |
| Sort then match | sort both sides, pair smallest with smallest | Assign Cookies, Boats to Save People |
| Last occurrence | extend the current part until the last occurrence | Partition Labels |

## Complexity

| Problem | Time | Space |
|---|---|---|
| Jump Game (55) | O(n) | O(1) |
| Jump Game II (45) | O(n) | O(1) |
| Gas Station (134) | O(n) | O(1) |
| Interval scheduling / 435 | O(n log n) for the sort | O(1) extra |
| Task Scheduler (621), formula | O(n) | O(1) (26 letters) |
| Partition Labels (763) | O(n) | O(1) (alphabet size) |
| Coin change by DP (for comparison) | O(amount · coins) | O(amount) |

## Template

Most greedy solutions look like: **(optional) sort by the right key, then one pass keeping a small running state**.

```ts
// Generic shape: sort by a key, scan once, keep a running "best so far".
function greedyTemplate<T>(items: T[], key: (x: T) => number, canTake: (x: T, state: number) => boolean, update: (x: T, state: number) => number, init: number): number {
  let state = init;
  let taken = 0;
  for (const x of items.toSorted((a, b) => key(a) - key(b))) {
    if (canTake(x, state)) {
      state = update(x, state);   // commit, never undo
      taken++;
    }
  }
  return taken;
}

// Concrete: interval scheduling, maximum number of non-overlapping intervals.
function maxNonOverlapping(intervals: Array<[number, number]>): number {
  let lastEnd = -Infinity, count = 0;
  for (const [s, e] of intervals.toSorted((a, b) => a[1] - b[1])) {
    if (s >= lastEnd) { count++; lastEnd = e; }
  }
  return count;
}
```

```python
def max_non_overlapping(intervals: list[list[int]]) -> int:
    last_end, count = float("-inf"), 0
    for s, e in sorted(intervals, key=lambda iv: iv[1]):   # sort by END
        if s >= last_end:                                   # greedy: earliest finish
            count += 1
            last_end = e
    return count
```

## Worked example

**[55. Jump Game](https://leetcode.com/problems/jump-game/)**: `nums[i]` is the max jump length from index `i`. Can you reach the last index from index 0?

Greedy insight: you do not need to choose *which* jump to take. Track `reach`, the farthest index reachable so far. If you ever stand on an index beyond `reach`, you are stuck.

Input `[2, 3, 1, 1, 4]`:

| i | nums[i] | i > reach? | reach = max(reach, i + nums[i]) |
|---|---|---|---|
| 0 | 2 | no | 2 |
| 1 | 3 | no | 4 |
| 2 | 1 | no | 4 |
| 3 | 1 | no | 4 |
| 4 | 4 | no | 8 |

Reached the end: **true**.

Input `[3, 2, 1, 0, 4]`:

| i | nums[i] | i > reach? | reach |
|---|---|---|---|
| 0 | 3 | no | 3 |
| 1 | 2 | no | 3 |
| 2 | 1 | no | 3 |
| 3 | 0 | no | 3 |
| 4 | 4 | **yes (4 > 3)** | stop |

Result: **false**. Every path funnels into index 3, which has jump length 0.

Why it is correct: `reach` is exactly the set `{0 .. reach}` of reachable indices (every index up to reach is reachable because you can take shorter jumps). So the answer is simply whether that interval covers `n - 1`.

```ts
function canJump(nums: number[]): boolean {
  let reach = 0;
  for (let i = 0; i < nums.length; i++) {
    if (i > reach) return false;          // stuck before i
    reach = Math.max(reach, i + nums[i]);
  }
  return true;
}
```

```python
def can_jump(nums: list[int]) -> bool:
    reach = 0
    for i, x in enumerate(nums):
        if i > reach:
            return False                  # stuck before i
        reach = max(reach, i + x)
    return True
```

## More templates

### Jump Game II: minimum jumps ([45](https://leetcode.com/problems/jump-game-ii/))

Implicit BFS: `[start, curEnd]` is the range reachable with `jumps` jumps; when you hit its end, you must jump again, and the new range ends at `farthest`.

```ts
function jump(nums: number[]): number {
  let jumps = 0, curEnd = 0, farthest = 0;
  for (let i = 0; i < nums.length - 1; i++) {
    farthest = Math.max(farthest, i + nums[i]);
    if (i === curEnd) { jumps++; curEnd = farthest; }
  }
  return jumps;
}
```

```python
def jump(nums: list[int]) -> int:
    jumps = cur_end = farthest = 0
    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])
        if i == cur_end:
            jumps += 1
            cur_end = farthest
    return jumps
```

### Gas Station ([134](https://leetcode.com/problems/gas-station/))

If total gas is at least total cost, a solution exists. Scan once; whenever the tank goes negative at `i`, no start in `[start, i]` can work (each of them arrives at `i + 1` with even less), so restart from `i + 1`.

```ts
function canCompleteCircuit(gas: number[], cost: number[]): number {
  let total = 0, tank = 0, start = 0;
  for (let i = 0; i < gas.length; i++) {
    const diff = gas[i] - cost[i];
    total += diff;
    tank += diff;
    if (tank < 0) { start = i + 1; tank = 0; }
  }
  return total >= 0 ? start : -1;
}
```

```python
def can_complete_circuit(gas: list[int], cost: list[int]) -> int:
    total = tank = start = 0
    for i, (g, c) in enumerate(zip(gas, cost)):
        total += g - c
        tank += g - c
        if tank < 0:
            start, tank = i + 1, 0
    return start if total >= 0 else -1
```

### Task Scheduler ([621](https://leetcode.com/problems/task-scheduler/))

The most frequent task (frequency `maxF`) forces `maxF - 1` full frames of length `n + 1`, plus a last frame holding every task tied at `maxF`. If there are enough other tasks to fill all idle slots, the answer is just the number of tasks.

```text
tasks = A A A B B B, n = 2
frames: [A B _] [A B _] [A B]    -> (3 - 1) * (2 + 1) + 2 = 8
```

```ts
function leastInterval(tasks: string[], n: number): number {
  const freq = new Array<number>(26).fill(0);
  for (const t of tasks) freq[t.charCodeAt(0) - 65]++;
  const maxF = Math.max(...freq);
  const countMax = freq.filter(f => f === maxF).length;
  return Math.max(tasks.length, (maxF - 1) * (n + 1) + countMax);
}
```

```python
from collections import Counter

def least_interval(tasks: list[str], n: int) -> int:
    freq = Counter(tasks).values()
    max_f = max(freq)
    count_max = sum(1 for f in freq if f == max_f)
    return max(len(tasks), (max_f - 1) * (n + 1) + count_max)
```

A max-heap simulation also works and generalizes to "print the actual schedule" (see [Top-K & Heaps pattern](/Software%20Engineering/DSA/04%20Patterns/Top-K%20%26%20Heaps%20pattern)).

### Partition Labels ([763](https://leetcode.com/problems/partition-labels/))

```ts
function partitionLabels(s: string): number[] {
  const last = new Map<string, number>();
  for (let i = 0; i < s.length; i++) last.set(s[i], i);
  const res: number[] = [];
  let start = 0, end = 0;
  for (let i = 0; i < s.length; i++) {
    end = Math.max(end, last.get(s[i])!);
    if (i === end) { res.push(end - start + 1); start = i + 1; }
  }
  return res;
}
```

```python
def partition_labels(s: str) -> list[int]:
    last = {c: i for i, c in enumerate(s)}
    res, start, end = [], 0, 0
    for i, c in enumerate(s):
        end = max(end, last[c])
        if i == end:
            res.append(end - start + 1)
            start = i + 1
    return res
```

## When to use it (recognition signals)

- **"Minimum number of ..." / "maximum number of ..."** where a simple sort order or running quantity seems to decide everything.
- **Intervals / scheduling** with "maximum non-overlapping", "minimum arrows", "minimum removals" (sort by end). See [Intervals](/Software%20Engineering/DSA/04%20Patterns/Intervals).
- **Reachability** in one dimension (jumps), **circular balance** (gas station), **frequency-driven scheduling** (task scheduler, reorganize string).
- Constraints hint at **O(n) or O(n log n)** with n up to 10^5 and no obvious subproblem table.
- **Red flag for greedy**: you can find a small counterexample where the locally best choice blocks a better global answer. Then use [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming) (coin change, 0/1 knapsack, longest increasing subsequence length via DP or patience sorting).

## Trade-offs & pitfalls

- **No proof, no greedy.** Greedy solutions often pass the examples and fail hidden tests. Before coding, try 3 or 4 tiny adversarial inputs, or compare against a brute force on random small inputs.
- **Wrong sort key.** Interval scheduling by start time or by shortest duration is wrong; by end time is right.
- **Fractional vs 0/1 knapsack.** Greedy by value/weight ratio is optimal for fractional knapsack and wrong for 0/1 knapsack.
- **Ties.** Decide how ties are broken and check it does not matter (or that it does).
- **Off-by-one in loops.** Jump Game II loops to `n - 2`; looping to `n - 1` can add an extra jump.
- **Overflow** in running sums is not an issue in Python or for typical JS ranges, but matters in Java/C++.
- **Greedy vs DP cost.** Greedy is O(n) where DP may be O(n²) or O(n · W). When both work, greedy wins; when unsure, DP is the safe fallback.

## Senior perspective

- **Greedy is everywhere in systems**: Huffman coding (compression in DEFLATE/gzip), Dijkstra's shortest paths, Prim's and Kruskal's minimum spanning trees, LRU-style cache eviction heuristics, load balancers picking the least-loaded server, OS schedulers choosing shortest remaining time, and TCP congestion control reacting to local signals.
- **Many production algorithms are greedy heuristics without optimality guarantees** (bin packing in container schedulers like Kubernetes, first-fit memory allocators). They trade optimality for speed and predictability; the senior skill is knowing the approximation bound and failure mode.
- **Matroids** are the theory behind "when greedy is optimal": if the feasible sets form a matroid, the greedy algorithm on sorted weights is optimal (CLRS covers this).
- **In review**, ask: what is the argument that the local choice is safe? What happens on ties, empty input, and adversarial ordering? Is there a test comparing against brute force?
- **In interviews**, say the greedy rule, then give a one-line exchange argument or invariant ("reach is the farthest reachable index"). If the interviewer pushes back, produce a counterexample check out loud and fall back to DP if needed.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 55 | [Jump Game](https://leetcode.com/problems/jump-game/) | Medium | track farthest reachable index |
| 45 | [Jump Game II](https://leetcode.com/problems/jump-game-ii/) | Medium | implicit BFS levels over reach windows |
| 134 | [Gas Station](https://leetcode.com/problems/gas-station/) | Medium | reset start when tank goes negative |
| 435 | [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | Medium | sort by end, earliest finish first |
| 621 | [Task Scheduler](https://leetcode.com/problems/task-scheduler/) | Medium | frames from the most frequent task |
| 763 | [Partition Labels](https://leetcode.com/problems/partition-labels/) | Medium | extend part to last occurrence |

Contrast: [322. Coin Change](https://leetcode.com/problems/coin-change/) (Medium) looks greedy but needs DP.

## Interview questions

<Accordion title="What two properties make a greedy algorithm correct?">
The greedy choice property (some optimal solution includes the greedy choice) and optimal substructure (after that choice, the remaining problem is a smaller instance whose optimal solution completes an optimal overall solution).
</Accordion>

<Accordion title="Sketch an exchange argument for interval scheduling by earliest end time.">
Take an optimal solution; replace its first interval with the interval that ends earliest overall. That interval ends no later, so it does not conflict with the rest, and the count is unchanged. Repeat on the remaining intervals. The greedy solution therefore has the same size as an optimal one.
</Accordion>

<Accordion title="Why does greedy fail for coin change? When does it work?">
Coins `[1, 3, 4]` and amount 6: greedy gives 4+1+1 (3 coins), optimal is 3+3 (2 coins). Committing to the largest coin can block a better combination. Greedy works for canonical systems such as US coins; in general use DP, O(amount · number of coins).
</Accordion>

<Accordion title="Gas Station: why can you skip all starts between the old start and i when the tank goes negative?">
Any start s in that range arrives at s with a non-negative tank from the earlier stations (or starts at 0), so starting later only removes non-negative contributions. If starting at `start` fails at `i`, starting at any s in `(start, i]` also fails at or before `i`. So the next candidate is `i + 1`.
</Accordion>

<Accordion title="How do you convince yourself a greedy idea is right under interview time pressure?">
Try small adversarial examples (ties, a long item vs many short ones, zeros), state an invariant, and give a short exchange argument. If a counterexample appears, switch to DP. In practice you can also write a brute force and compare on random tiny inputs.
</Accordion>

<Accordion title="Task Scheduler: explain the formula.">
The most frequent task appears `maxF` times and needs at least `n` slots between occurrences, forming `maxF - 1` frames of length `n + 1` plus a final partial frame containing every task tied at `maxF`. If other tasks overflow the idle slots, no idling is needed and the answer is `len(tasks)`. So `max(len(tasks), (maxF - 1)(n + 1) + countMax)`.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming) (when greedy fails) · [Intervals](/Software%20Engineering/DSA/04%20Patterns/Intervals) · [Top-K & Heaps pattern](/Software%20Engineering/DSA/04%20Patterns/Top-K%20%26%20Heaps%20pattern)
- [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs) (Dijkstra, Prim, Kruskal) · [Union-Find](/Software%20Engineering/DSA/02%20Data%20Structures/Union-Find) (Kruskal) · [Sorting Algorithms](/Software%20Engineering/DSA/03%20Algorithms/Sorting%20Algorithms)
- [Problem Solving](/Software%20Engineering/DSA/01%20Foundations/Problem%20Solving) · [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis)

## Sources
- Cormen et al., *Introduction to Algorithms*, 4th ed., MIT Press 2022 (Greedy Algorithms chapter: activity selection, Huffman codes, matroids)
- Kleinberg & Tardos, *Algorithm Design*, Pearson 2005, ch. 4 Greedy Algorithms (exchange arguments, "greedy stays ahead")
- [Tech Interview Handbook: Algorithms study cheatsheet](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/)
- [LeetCode's Interview Crash Course](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/)
- [NeetCode roadmap](https://neetcode.io/roadmap)
