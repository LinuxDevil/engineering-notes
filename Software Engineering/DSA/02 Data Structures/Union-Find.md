---
title: "Union-Find"
tags: [dsa, dsa/data-structures]
aliases: [Union Find, UnionFind, DSU, Disjoint Set Union, Disjoint Set, Disjoint-Set Forest]
updated: 2026-09-27
---
# Union-Find

<Note>
**TL;DR**

**Union-Find** (Disjoint Set Union, DSU) tracks a collection of non-overlapping groups and supports two operations: `find(x)` (which group is x in?) and `union(a, b)` (merge their groups). Each group is a tree stored in a `parent` array; the root is the group's representative. With **path compression** plus **union by rank or size**, both operations run in amortized **O(alpha(n))**, where alpha is the inverse Ackermann function, at most 4 for any realistic n. Use it for dynamic connectivity: connected components, cycle detection in undirected graphs, and Kruskal's MST.
</Note>

## The idea

Picture every element as a person holding a pointer to a "boss". Follow bosses upward until you reach someone who is their own boss: that is the group leader. Two people are in the same group exactly when they have the same leader. Merging two groups just means making one leader report to the other.

```text
parent = [0, 0, 1, 3, 3, 5]        Groups as trees:

index:    0  1  2  3  4  5             0        3      5
                                        |        |
                                        1        4
                                        |
                                        2
find(2) -> 1 -> 0 (root)   find(4) -> 3   find(5) -> 5
same(2, 4)? roots 0 and 3 -> no
```

It answers "are these connected?" fast while edges keep arriving, which BFS/DFS cannot do without re-traversing ([Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs)). It cannot efficiently **split** groups or delete edges.

## How it works

### Naive version

- `find(x)`: walk `parent` until `parent[x] == x`.
- `union(a, b)`: `parent[find(a)] = find(b)`.

Problem: unions in a bad order build a chain, so `find` becomes O(n).

### Optimization 1: union by rank or size

Always attach the **smaller** tree under the root of the **larger** one.

- **By size**: track node count per root. Also gives you component sizes for free (handy in many problems).
- **By rank**: track an upper bound on tree height; only increases when two equal-rank trees merge.

Either one alone keeps height at O(log n): a node's depth grows only when its tree at least doubles in size.

### Optimization 2: path compression

During `find`, point every node on the path directly at the root, so the next lookup is one hop.

```text
Before find(D):      After find(D) with full compression:
    A                        A
    |                     /  |  \
    B                    B   C   D
    |
    C
    |
    D
```

Variants: **full compression** (two passes or recursion) and **path halving** (`parent[x] = parent[parent[x]]` while walking, one pass, no recursion). Both give the same bound.

### Both together: inverse Ackermann

Tarjan (1975) proved that with both optimizations, m operations on n elements take O(m * alpha(n)) time. alpha(n) grows so slowly that it is at most 4 for any n up to a number vastly larger than the count of atoms in the observable universe (CLRS ch. 19). In practice, treat each operation as **amortized constant time**. With only one of the two optimizations you get O(log n) amortized per operation.

## Complexity

| Operation | Naive | Rank/size only | Path compression only | Both |
|---|---|---|---|---|
| `find` | O(n) | O(log n) | O(log n) amortized | O(alpha(n)) amortized |
| `union` | O(n) | O(log n) | O(log n) amortized | O(alpha(n)) amortized |
| `connected(a, b)` | O(n) | O(log n) | O(log n) amortized | O(alpha(n)) amortized |
| Build (n singletons) | O(n) | O(n) | O(n) | O(n) |
| Space | O(n) | O(n) | O(n) | O(n) |

Amortized means a single call can be slower (the first `find` on a long path), but the average over any sequence is bounded.

## Implementation

### DSU with path compression and union by size

```ts
class DSU {
  private parent: number[];
  private size: number[];
  count: number; // number of components

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.size = new Array(n).fill(1);
    this.count = n;
  }

  find(x: number): number {
    while (this.parent[x] !== x) {
      this.parent[x] = this.parent[this.parent[x]]; // path halving
      x = this.parent[x];
    }
    return x;
  }

  /** Returns false if a and b were already connected. */
  union(a: number, b: number): boolean {
    let ra = this.find(a), rb = this.find(b);
    if (ra === rb) return false;
    if (this.size[ra] < this.size[rb]) [ra, rb] = [rb, ra]; // ra is the larger
    this.parent[rb] = ra;
    this.size[ra] += this.size[rb];
    this.count--;
    return true;
  }

  connected(a: number, b: number): boolean {
    return this.find(a) === this.find(b);
  }

  componentSize(x: number): number {
    return this.size[this.find(x)];
  }
}

const d = new DSU(6);
d.union(0, 1); d.union(1, 2); d.union(3, 4);
console.log(d.connected(0, 2), d.connected(2, 4), d.count); // true false 3
```

```python
class DSU:
    def __init__(self, n: int) -> None:
        self.parent = list(range(n))
        self.size = [1] * n
        self.count = n  # number of components

    def find(self, x: int) -> int:
        # iterative path halving: no recursion limit issues
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]
            x = self.parent[x]
        return x

    def union(self, a: int, b: int) -> bool:
        """Returns False if a and b were already connected."""
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False
        if self.size[ra] < self.size[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra
        self.size[ra] += self.size[rb]
        self.count -= 1
        return True

    def connected(self, a: int, b: int) -> bool:
        return self.find(a) == self.find(b)


d = DSU(6)
d.union(0, 1); d.union(1, 2); d.union(3, 4)
print(d.connected(0, 2), d.connected(2, 4), d.count)  # True False 3
```

<Tip>
**Keys that are not 0..n-1**

For strings or sparse ids, use a hash map for `parent` and create entries on first sight (`parent.setdefault(x, x)` in Python, `if (!parent.has(x)) parent.set(x, x)` in TS), or map each key to an integer id first, as in Accounts Merge below.
</Tip>

The recursive full-compression `find` is shorter (`if p[x] != x: p[x] = find(p[x])`) but can hit Python's recursion limit on the first call over a long chain if you do not also use union by size.

## Applications

### Connected components (LC 547 Number of Provinces)

Union every connected pair; the answer is `dsu.count`.

```ts
function findCircleNum(isConnected: number[][]): number {
  const n = isConnected.length, dsu = new DSU(n);
  for (let i = 0; i < n; i++)
    for (let j = i + 1; j < n; j++)
      if (isConnected[i][j]) dsu.union(i, j);
  return dsu.count;
}
```

```python
def find_circle_num(is_connected: list[list[int]]) -> int:
    n = len(is_connected)
    dsu = DSU(n)
    for i in range(n):
        for j in range(i + 1, n):
            if is_connected[i][j]:
                dsu.union(i, j)
    return dsu.count
```

### Cycle detection: Redundant Connection (LC 684)

A tree with n nodes plus one extra edge; return the edge that creates the cycle (the last one in input order). If both endpoints already share a root, that edge closes a cycle.

Trace for `edges = [[1,2], [1,3], [2,3]]` (nodes are 1-indexed, so use `DSU(n + 1)`):

| Edge | find(u) | find(v) | Action | Groups |
|---|---|---|---|---|
| [1,2] | 1 | 2 | union | \{1,2\} \{3\} |
| [1,3] | 1 | 3 | union | \{1,2,3\} |
| [2,3] | 1 | 1 | same root: **answer** | |

```ts
function findRedundantConnection(edges: number[][]): number[] {
  const dsu = new DSU(edges.length + 1);
  for (const [u, v] of edges) if (!dsu.union(u, v)) return [u, v];
  return [];
}
```

```python
def find_redundant_connection(edges: list[list[int]]) -> list[int]:
    dsu = DSU(len(edges) + 1)
    for u, v in edges:
        if not dsu.union(u, v):
            return [u, v]
    return []
```

### Kruskal's MST

Sort edges by weight and add an edge only if `union` returns true. The DSU is what makes Kruskal O(E log E): the sort dominates. Full code in [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs).

### Accounts Merge (LC 721)

Each account is `[name, email1, email2, ...]`. Accounts sharing any email belong to the same person. Union all emails within an account, then group emails by root.

```ts
function accountsMerge(accounts: string[][]): string[][] {
  const id = new Map<string, number>();
  const owner = new Map<string, string>();
  for (const [name, ...emails] of accounts)
    for (const e of emails) {
      if (!id.has(e)) id.set(e, id.size);
      owner.set(e, name);
    }

  const dsu = new DSU(id.size);
  for (const [, first, ...rest] of accounts)
    for (const e of rest) dsu.union(id.get(first)!, id.get(e)!);

  const groups = new Map<number, string[]>();
  for (const [e, i] of id) {
    const r = dsu.find(i);
    if (!groups.has(r)) groups.set(r, []);
    groups.get(r)!.push(e);
  }
  return [...groups.values()].map((es) => [owner.get(es[0])!, ...es.sort()]);
}
```

```python
from collections import defaultdict

def accounts_merge(accounts: list[list[str]]) -> list[list[str]]:
    email_id: dict[str, int] = {}
    owner: dict[str, str] = {}
    for name, *emails in accounts:
        for e in emails:
            email_id.setdefault(e, len(email_id))
            owner[e] = name

    dsu = DSU(len(email_id))
    for _, first, *rest in accounts:
        for e in rest:
            dsu.union(email_id[first], email_id[e])

    groups: dict[int, list[str]] = defaultdict(list)
    for e, i in email_id.items():
        groups[dsu.find(i)].append(e)
    return [[owner[es[0]]] + sorted(es) for es in groups.values()]
```

Complexity: O(N * alpha(N) + N log N) where N is the total number of emails; the sort dominates.

### Other uses

- **Equations with equality and inequality** (LC 990): union all `==` pairs, then check no `!=` pair shares a root.
- **Network repair** (LC 1319): extra cables = redundant edges; answer = components - 1 if enough cables.
- **Weighted Union-Find** (LC 399 Evaluate Division): store the ratio to the parent on each edge; multiply along the path during `find`.
- **Grid percolation / islands arriving online** (LC 305 is premium): add cells over time and count components after each addition.
- **Offline dynamic connectivity**: process queries in a clever order (e.g. reverse time) so you only ever union.

## Trade-offs & pitfalls

- **Comparing `parent[a] == parent[b]`** instead of `find(a) == find(b)`. Parents are not roots after arbitrary unions.
- **Forgetting to union the roots**: `parent[a] = b` instead of `parent[find(a)] = find(b)` corrupts the forest.
- **1-indexed nodes** (LC 684, many graph problems): allocate `n + 1`.
- **Recursion depth** in recursive `find` on a long chain; use the iterative version.
- **No deletions or splits**: DSU only merges. If edges get removed, think offline in reverse, or use a different structure (link-cut trees, or recompute).
- **Directed graphs**: DSU tracks undirected connectivity. For directed cycles use DFS or Kahn's topological sort ([Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs)).
- **When BFS/DFS is simpler**: for a static graph answered once, a single traversal is O(V + E) and just as good. DSU shines when edges arrive incrementally or you need many connectivity queries interleaved with merges.

## Senior perspective

- **Where it is used for real**: Kruskal's MST in network design and clustering, image processing (connected-component labeling in computer vision, e.g. OpenCV's `connectedComponents`), type inference and unification in compilers (Hindley-Milner uses union-find over type variables), alias analysis, deduplication and entity resolution (merge customer records that share an email or phone, which is Accounts Merge at scale), percolation simulations, and maze generation.
- **At scale**: in distributed systems the same idea appears as connected-components jobs in Spark GraphX or iterative label propagation, since a single in-memory parent array does not fit. Entity resolution pipelines often batch unions and then materialize a `cluster_id` column.
- **Memory**: two integer arrays, extremely cache-friendly when keys are dense ints. In JS, `Int32Array` for `parent` and `size` avoids boxed numbers and keeps V8 on packed element kinds; in Python, remapping keys to ints avoids dict overhead.
- **Interview framing**: mention both optimizations and the alpha(n) bound, say "effectively constant", explain when you would choose DSU over BFS (incremental edges, online queries), and write the ~20-line class from memory. Returning a boolean from `union` makes cycle detection and Kruskal one-liners.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 1971 | [Find if Path Exists in Graph](https://leetcode.com/problems/find-if-path-exists-in-graph/) | Easy | Union all edges, compare roots |
| 547 | [Number of Provinces](https://leetcode.com/problems/number-of-provinces/) | Medium | Count components |
| 684 | [Redundant Connection](https://leetcode.com/problems/redundant-connection/) | Medium | First edge whose endpoints share a root |
| 721 | [Accounts Merge](https://leetcode.com/problems/accounts-merge/) | Medium | Union emails, group by root |
| 990 | [Satisfiability of Equality Equations](https://leetcode.com/problems/satisfiability-of-equality-equations/) | Medium | Union `==`, then verify `!=` |
| 1319 | [Number of Operations to Make Network Connected](https://leetcode.com/problems/number-of-operations-to-make-network-connected/) | Medium | Components - 1, if edges >= n - 1 |
| 399 | [Evaluate Division](https://leetcode.com/problems/evaluate-division/) | Medium | Weighted union-find (or graph DFS) |
| 1584 | [Min Cost to Connect All Points](https://leetcode.com/problems/min-cost-to-connect-all-points/) | Medium | Kruskal with DSU |

## Interview questions

<Accordion title="What problem does Union-Find solve?">
Dynamic connectivity over disjoint sets: merge two groups and ask whether two elements are in the same group, with edges arriving over time. Typical uses are connected components, undirected cycle detection, Kruskal's MST, and grouping/deduplication.
</Accordion>

<Accordion title="Explain path compression and union by rank/size.">
Path compression makes every node visited during `find` point closer to (or directly at) the root, flattening the tree for future calls. Union by rank/size attaches the shorter or smaller tree under the other root so trees stay shallow. Together they give amortized O(alpha(n)) per operation.
</Accordion>

<Accordion title="What is alpha(n) and why is it &quot;effectively constant&quot;?">
The inverse Ackermann function. The Ackermann function grows so fast that its inverse is at most 4 for any input size that could exist in practice, so the amortized cost per operation is bounded by a small constant.
</Accordion>

<Accordion title="When would you use Union-Find instead of BFS/DFS?">
When edges are added incrementally and you need connectivity answers between additions, when you process edges in a specific order (Kruskal, sorted by weight), or when you only need grouping, not paths. For a single question on a static graph, BFS/DFS is equally good and gives paths too.
</Accordion>

<Accordion title="How do you detect a cycle in an undirected graph with Union-Find?">
Process each edge (u, v): if `find(u) == find(v)` they are already connected, so this edge closes a cycle; otherwise union them. This is exactly LC 684.
</Accordion>

<Accordion title="Can Union-Find handle edge deletions?">
Not directly; it only merges. Options: process operations offline in reverse (deletions become unions), use a rollback DSU (union by rank without path compression, undo via a stack) for divide-and-conquer offline algorithms, or use more complex dynamic connectivity structures.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs) (Kruskal, components, cycle detection) · [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns)
- [Trees & Binary Search Trees](/Software%20Engineering/DSA/02%20Data%20Structures/Trees%20%26%20Binary%20Search%20Trees) (DSU is a forest of trees) · [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) (non-integer keys)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis) (amortized analysis)

## Sources
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022: ch. 19 (Data structures for disjoint sets)
- Tarjan, R. E. (1975). "Efficiency of a Good But Not Linear Set Union Algorithm". *Journal of the ACM* 22(2): 215–225. [doi:10.1145/321879.321884](https://doi.org/10.1145/321879.321884)
- [Sedgewick & Wayne, Algorithms 4th ed.: 1.5 Case Study: Union-Find](https://algs4.cs.princeton.edu/15uf/)
- [cp-algorithms: Disjoint Set Union](https://cp-algorithms.com/data_structures/disjoint_set_union.html)
- [OpenCV docs: connectedComponents](https://docs.opencv.org/4.x/d3/dc0/group__imgproc__shape.html)
- [LeetCode Interview Crash Course](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/)
