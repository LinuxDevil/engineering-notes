---
tags: [interviews, interviews/coding]
aliases: [Interview 1, Codility Assessment, Online Assessment Java]
updated: 2026-09-27
---
# Online Coding Assessment - Bit Count, Consecutive Path & Missing Rolls

> [!tldr] TL;DR
> A real timed online assessment (Codility style) with three algorithm tasks solved in Java, plus Git and shell multiple-choice questions. The pattern to learn: read the constraints first (they tell you about overflow and the target complexity), reduce the problem to something simple, and test the given examples plus edge cases. JavaScript versions of the same solutions are in [[JavaScript Interview Questions]].

> [!info] How these tests are scored
> Codility-style platforms score **correctness** (hidden edge-case tests) and **performance** (large inputs with a time limit). A brute force that passes the examples can still score 30%. Always check: empty inputs, minimum/maximum values, overflow, duplicates.

## Task 1: Count the set bits of A × B

**Problem.** Given two non-negative integers A and B (each in `[0..100,000,000]`), return the number of bits set to 1 in the binary representation of A × B.

Example: A = 3, B = 7 → 21 = `10101`₂ → **3**.
> **(fixed)** The original text wrote 21 as `101010`; that is 42. 21 is `10101`.

**Key insight from the constraints.** 10⁸ × 10⁸ = 10¹⁶, which overflows a 32-bit `int` (max ≈ 2.1 × 10⁹) but fits in a 64-bit `long` (max ≈ 9.2 × 10¹⁸). Cast **before** multiplying.

```java
public class Main {
    public static void main(String[] args) {
        System.out.println(countBits(3, 7)); // 3
    }

    public static int countBits(int A, int B) {
        long product = (long) A * B;   // cast first, otherwise int overflow happens before widening
        return Long.bitCount(product); // population count, no string conversion needed
    }
}
```

Manual version (what the interviewer may ask for), using Kernighan's trick: `x & (x - 1)` clears the lowest set bit, so the loop runs once per set bit.

```java
static int popcount(long x) {
    int count = 0;
    while (x != 0) { x &= x - 1; count++; }
    return count;
}
```

Complexity: O(1) (at most 64 iterations). Edge cases: A or B = 0 → 0.

## Task 2: Path 1 → N through every vertex in increasing order

**Problem.** An undirected graph with N vertices (1..N) and M edges given by arrays A and B. Return `true` if there is a path 1 → 2 → 3 → … → N using direct edges only. N, M ≤ 100,000, no self-loops, no duplicate edges.

Examples: N = 4, A = [1,2,4,4,3], B = [2,3,1,3,1] → `true` (edges 1–2, 2–3, 3–4). N = 4, A = [1,2,1,3], B = [2,4,3,4] → `false`. N = 6, A = [2,4,5,3], B = [3,5,6,4] → `false` (no edge 1–2).

**Key insight.** No graph search is needed. The path exists **if and only if** every edge `(i, i+1)` for i = 1..N-1 exists. Because edges are undirected, normalise each edge so the smaller end comes first.

```java
public static boolean hasPath(int N, int[] A, int[] B) {
    boolean[] next = new boolean[N + 1];      // next[i] = edge (i, i+1) exists
    for (int k = 0; k < A.length; k++) {
        int lo = Math.min(A[k], B[k]), hi = Math.max(A[k], B[k]);
        if (hi == lo + 1) next[lo] = true;
    }
    for (int i = 1; i < N; i++) if (!next[i]) return false;
    return true;
}
```

The original solution (kept below) does the same with an `int[]` storing the neighbour and a `HashMap` variant. The array version is better: O(N + M) time, no boxing, no hashing.

> [!example]- Original solutions (array and HashMap)
> ```java
> public static boolean hasPath(int N, int[] A, int[] B) {
>     int[] edges = fillEdges(N, A, B);
>     return checkPath(N, edges);
> }
> private static int[] fillEdges(int N, int[] A, int[] B) {
>     int[] edges = new int[N + 1];
>     for (int i = 0; i < A.length; i++) {
>         if (A[i] + 1 == B[i]) edges[A[i]] = B[i];
>         else if (B[i] + 1 == A[i]) edges[B[i]] = A[i];
>     }
>     return edges;
> }
> private static boolean checkPath(int N, int[] edges) {
>     for (int i = 1; i < N; i++) if (edges[i] != i + 1) return false;
>     return true;
> }
>
> // HashMap version: same logic, slower constant factor
> private static HashMap<Integer, Integer> fillEdgesMap(int N, int[] A, int[] B) {
>     HashMap<Integer, Integer> edges = new HashMap<>();
>     for (int i = 0; i < A.length; i++) {
>         if (A[i] + 1 == B[i]) edges.put(A[i], B[i]);
>         else if (B[i] + 1 == A[i]) edges.put(B[i], A[i]);
>     }
>     return edges;
> }
> private static boolean checkPathMap(int N, HashMap<Integer, Integer> edges) {
>     for (int i = 1; i < N; i++) if (!edges.getOrDefault(i, -1).equals(i + 1)) return false;
>     return true;
> }
> ```

## Task 3: Missing dice rolls

**Problem.** You remember N rolls in array A, forgot F rolls, and know the mean of all N + F rolls is M. Return any F values in 1..6 that make the mean exactly M, or `[0]` if impossible. N, F ≤ 100,000.

Examples: A = [3,2,4,3], F = 2, M = 4 → [6,6]. A = [1,5,6], F = 4, M = 3 → e.g. [6,1,1,1]. A = [6,1], F = 1, M = 1 → [0].

**Key insight.** The required sum of the missing rolls is `need = M × (N + F) − sum(A)`. A solution exists if and only if `F ≤ need ≤ 6F`. Then start every roll at 1 and greedily add up to 5 to each until `need` is used up (or spread evenly: `need / F` each plus 1 for the first `need % F`).

```java
import java.util.Arrays;

public static int[] findMissingRolls(int[] A, int F, int M) {
    int need = M * (A.length + F);            // ≤ 6 × 200,000: fits in int
    for (int r : A) need -= r;
    if (need < F || need > 6 * F) return new int[]{0};

    int[] rolls = new int[F];
    int base = need / F, extra = need % F;    // even spread, O(F)
    for (int i = 0; i < F; i++) rolls[i] = base + (i < extra ? 1 : 0);
    return rolls;
}
```

The original greedy (fill with 1s, then add `min(5, remaining)` to each) is also correct and O(N + F); the even spread is shorter.

## Git and shell multiple choice

| Question | Answer | Why |
|---|---|---|
| A commit's SHA does **not** change if you change only: author / date / message / none | **None** | Author, committer, dates, message, tree and parent ids are all hashed into the commit object. Change any and you get a new SHA (that's why `git commit --amend` and rebase rewrite history). |
| Who last modified a line? `git blame` / `fetch` / `log` / `diff` | **`git blame <file>`** | `git log -L 10,20:file` shows the full history of a line range. |
| Which prints "hello" to stdout? `cat hello` / `echo hello` / `ls hello` / `cp hello` | **`echo hello`** | `cat hello` prints the contents of a *file* named hello. |
| Replace `...` so the loop runs every `.py` file in the tree: `for ... do python $i; done` | **B: ``i in `find . -name '*.py'` ``** | In `for` the variable name has no `$`; `[ ]` is not command substitution. Note the original option was missing its closing quote. |

Safer version that handles spaces in file names:
```bash
find . -name '*.py' -exec python {} \;
# or
find . -name '*.py' -print0 | while IFS= read -r -d '' f; do python "$f"; done
```

## Interview questions
> [!question]- Why cast to `long` before multiplying, not after?
> `(long)(A * B)` multiplies two `int`s first; the overflow already happened and the wrong value is widened. `(long) A * B` promotes A, so the multiplication is done in 64 bits.

> [!question]- How do you count set bits without a library call?
> Kernighan: `while (x != 0) { x &= x - 1; count++; }`. Each step clears the lowest set bit, so it runs once per 1-bit. Hardware `POPCNT` is what `Long.bitCount` compiles to on modern JVMs.

> [!question]- Why is no BFS/DFS needed for the path task?
> The path must visit 1, 2, …, N in order with direct edges, so the only possible path is fixed. It exists exactly when each consecutive edge exists, which is a set-membership check.

> [!question]- What is the feasibility condition in the dice task?
> Each missing roll is between 1 and 6, so their sum is between F and 6F. If the required sum lies in that range, some combination works (greedy or even spread proves it).

> [!question]- How do you approach a timed online assessment?
> Read constraints first to find the target complexity and overflow risks; write the simplest correct solution; test the examples and edge cases (0, 1, max values, empty arrays); only then optimise. Leave time to re-read the problem statement.

## Related
- [[Interviews Index]]
- [[DSA Index]] · [[JavaScript Interview Questions]] (JS versions of these tasks)
- [[Interview Process & Preparation Plan]]

## Sources
- [Codility: How are tests scored](https://support.codility.com/hc/en-us)
- [Java `Long.bitCount` API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Long.html#bitCount(long))
- [Git documentation: git-blame](https://git-scm.com/docs/git-blame), [Git internals: Git objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- Sean Eron Anderson, [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html#CountBitsSetKernighan)
