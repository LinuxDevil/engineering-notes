---
title: "Graphs"
tags: [dsa, dsa/data-structures]
aliases: [Graph, Graph Theory, Adjacency List, Adjacency Matrix, Topological Sort, Dijkstra, Kruskal, Prim, MST]
updated: 2026-09-27
---
# Graphs

<Note>
**TL;DR**

A **graph** is a set of vertices connected by edges. Edges can be directed or undirected, weighted or unweighted. Store sparse graphs as an **adjacency list** (the default), dense ones as a **matrix**, and use an **edge list** when you process edges globally (Kruskal, Bellman-Ford). Core algorithms: **BFS/DFS** for reachability, **topological sort** for dependency ordering, **Dijkstra** for shortest paths with non-negative weights, **Bellman-Ford** when weights can be negative, and **Kruskal/Prim** for minimum spanning trees.
</Note>

## The idea

Graphs model relationships: people and friendships, cities and roads, packages and dependencies, services and network calls, web pages and links. Trees ([Trees & Binary Search Trees](/Software%20Engineering/DSA/02%20Data%20Structures/Trees%20%26%20Binary%20Search%20Trees)) and linked lists are special cases: a tree is a connected graph with no cycles.

{/* diagram source: /diagrams/diagram-186.mmd */}
![diagram](/diagrams/diagram-186.svg)

This is a **directed, weighted** graph. The shortest path from A to D is `A -> C -> B -> D` with cost 1 + 2 + 1 = 4, even though `A -> B -> D` uses fewer edges.

### Vocabulary

| Term | Meaning |
|---|---|
| Vertex (node), edge | V = vertices, E = edges. Complexity is stated in terms of V and E |
| Directed / undirected | One-way arrows vs two-way links |
| Weighted | Edges carry a cost, distance or capacity |
| Degree | Number of edges at a vertex; directed graphs split into in-degree and out-degree |
| Path, cycle | Sequence of edges; a cycle returns to its start |
| DAG | Directed acyclic graph: dependencies, build systems, task pipelines |
| Connected component | Maximal set of vertices reachable from each other (undirected) |
| Sparse vs dense | E close to V vs E close to V^2 |
| Implicit graph | Edges are computed, not stored: grid cells, word ladder, game states |

## Representations

Same undirected graph with edges `0-1, 0-2, 1-2, 2-3`:

| Representation | Example | Space | Check edge (u, v) | Iterate neighbors of u | Best for |
|---|---|---|---|---|---|
| **Adjacency list** | `{0: [1, 2], 1: [0, 2], 2: [0, 1, 3], 3: [2]}` | O(V + E) | O(deg u) (O(1) with sets) | O(deg u) | Most problems, sparse graphs |
| **Adjacency matrix** | `m[u][v] = 1` in a V x V grid | O(V^2) | O(1) | O(V) | Dense graphs, Floyd-Warshall, small V |
| **Edge list** | `[[0,1], [0,2], [1,2], [2,3]]` | O(E) | O(E) | O(E) | Kruskal, Bellman-Ford, input format |

LeetCode usually hands you an edge list or `n` plus `edges`; the first step is almost always to build an adjacency list.

```ts
function buildAdj(n: number, edges: number[][], directed = false): number[][] {
  const adj: number[][] = Array.from({ length: n }, () => []);
  for (const [u, v] of edges) {
    adj[u].push(v);
    if (!directed) adj[v].push(u);
  }
  return adj;
}
```

```python
from collections import defaultdict

def build_adj(n: int, edges: list[list[int]], directed: bool = False) -> list[list[int]]:
    adj = [[] for _ in range(n)]
    for u, v in edges:
        adj[u].append(v)
        if not directed:
            adj[v].append(u)
    return adj

# weighted: adj[u].append((v, w)); non-integer ids: adj = defaultdict(list)
```

## How it works

### BFS and DFS (briefly)

- **BFS** explores level by level with a queue. On an **unweighted** graph it finds shortest paths in number of edges.
- **DFS** goes deep first with recursion or a stack. Used for connectivity, cycle detection, path enumeration, and topological order (post-order).
- Both are O(V + E). Always keep a `visited` set, since graphs (unlike trees) have cycles.

Problem-level templates (islands, multi-source BFS, grids) live in [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns); the tree versions are in [Tree Traversal - DFS & BFS](/Software%20Engineering/DSA/03%20Algorithms/Tree%20Traversal%20-%20DFS%20%26%20BFS).

```ts
function bfs(adj: number[][], start: number): number[] {
  const dist = new Array(adj.length).fill(-1);
  dist[start] = 0;
  const queue = [start];
  for (let head = 0; head < queue.length; head++) { // index pointer: O(1) dequeue
    const u = queue[head];
    for (const v of adj[u]) {
      if (dist[v] === -1) {
        dist[v] = dist[u] + 1;
        queue.push(v);
      }
    }
  }
  return dist; // edges from start, -1 if unreachable
}
```

```python
from collections import deque

def bfs(adj: list[list[int]], start: int) -> list[int]:
    dist = [-1] * len(adj)
    dist[start] = 0
    q = deque([start])
    while q:
        u = q.popleft()
        for v in adj[u]:
            if dist[v] == -1:
                dist[v] = dist[u] + 1
                q.append(v)
    return dist
```

### Topological sort (Kahn's algorithm)

Order the vertices of a DAG so every edge `u -> v` has u before v (build order, course prerequisites, spreadsheet recalculation).

1. Compute in-degree of every vertex.
2. Queue every vertex with in-degree 0.
3. Pop u, append to the order, decrement in-degree of each neighbor; enqueue neighbors that hit 0.
4. If the order contains fewer than V vertices, there is a **cycle**.

```ts
function topoSort(n: number, edges: number[][]): number[] | null {
  const adj: number[][] = Array.from({ length: n }, () => []);
  const indeg = new Array(n).fill(0);
  for (const [u, v] of edges) { adj[u].push(v); indeg[v]++; }

  const order: number[] = [];
  for (let u = 0; u < n; u++) if (indeg[u] === 0) order.push(u);
  for (let head = 0; head < order.length; head++) { // order doubles as the queue
    for (const v of adj[order[head]]) if (--indeg[v] === 0) order.push(v);
  }
  return order.length === n ? order : null; // null means cycle
}
```

```python
from collections import deque

def topo_sort(n: int, edges: list[list[int]]) -> list[int] | None:
    adj = [[] for _ in range(n)]
    indeg = [0] * n
    for u, v in edges:
        adj[u].append(v)
        indeg[v] += 1
    q = deque(u for u in range(n) if indeg[u] == 0)
    order = []
    while q:
        u = q.popleft()
        order.append(u)
        for v in adj[u]:
            indeg[v] -= 1
            if indeg[v] == 0:
                q.append(v)
    return order if len(order) == n else None
```

For LC 207 Course Schedule, the edge `[a, b]` means "take b before a", so the edge is `b -> a`. Getting that direction wrong is the classic bug. The alternative is DFS with three colors (unvisited, in progress, done): an edge to an "in progress" node is a cycle, and reverse post-order is a topological order.

### Dijkstra (single-source shortest paths, non-negative weights)

Greedy: repeatedly settle the unvisited vertex with the smallest known distance, using a min-heap. Uses lazy deletion: stale heap entries are skipped when popped.

Trace on the graph above from A (A=0, B=1, C=2, D=3):

| Pop | dist A | dist B | dist C | dist D | Heap after |
|---|---|---|---|---|---|
| start | 0 | inf | inf | inf | (0,A) |
| (0,A) | 0 | 4 | 1 | inf | (1,C) (4,B) |
| (1,C) | 0 | **3** | 1 | 6 | (3,B) (4,B) (6,D) |
| (3,B) | 0 | 3 | 1 | **4** | (4,B) (4,D) (6,D) |
| (4,B) | stale, skip | | | | (4,D) (6,D) |
| (4,D) | 0 | 3 | 1 | 4 | (6,D) then stale |

TypeScript needs a heap; this uses the `Heap` class from [Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues).

```ts
// import { Heap } from "./heap"; see Heaps & Priority Queues
function dijkstra(n: number, edges: [number, number, number][], src: number): number[] {
  const adj: [number, number][][] = Array.from({ length: n }, () => []);
  for (const [u, v, w] of edges) adj[u].push([v, w]);

  const dist = new Array(n).fill(Infinity);
  dist[src] = 0;
  const pq = new Heap<[number, number]>((a, b) => a[0] < b[0]); // [dist, node]
  pq.push([0, src]);
  while (pq.size) {
    const [d, u] = pq.pop()!;
    if (d > dist[u]) continue; // stale entry
    for (const [v, w] of adj[u]) {
      if (d + w < dist[v]) {
        dist[v] = d + w;
        pq.push([dist[v], v]);
      }
    }
  }
  return dist;
}
console.log(dijkstra(4, [[0, 1, 4], [0, 2, 1], [2, 1, 2], [1, 3, 1], [2, 3, 5]], 0)); // [0, 3, 1, 4]
```

```python
import heapq

def dijkstra(n: int, edges: list[tuple[int, int, int]], src: int) -> list[float]:
    adj = [[] for _ in range(n)]
    for u, v, w in edges:
        adj[u].append((v, w))
    dist = [float("inf")] * n
    dist[src] = 0
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue  # stale entry
        for v, w in adj[u]:
            if d + w < dist[v]:
                dist[v] = d + w
                heapq.heappush(pq, (dist[v], v))
    return dist

print(dijkstra(4, [(0, 1, 4), (0, 2, 1), (2, 1, 2), (1, 3, 1), (2, 3, 5)], 0))  # [0, 3, 1, 4]
```

### Bellman-Ford (negative weights)

Relax **every** edge V - 1 times. O(V * E). If a V-th pass still improves a distance, there is a **negative cycle** reachable from the source. Use it when weights can be negative (currency arbitrage, some cost models), or when the problem limits the number of edges: LC 787 runs k + 1 rounds of relaxation, copying the distance array each round so one round uses at most one more edge.

Other shortest-path tools: **0-1 BFS** (deque, weights 0 or 1), **Floyd-Warshall** (all pairs, O(V^3), small V), **A*** (Dijkstra plus a heuristic, used in games and maps).

### Minimum spanning tree (MST)

For a connected, undirected, weighted graph, the MST is the set of V - 1 edges that connects all vertices with minimum total weight (network cabling, clustering, approximating TSP).

- **Kruskal**: sort edges by weight; add each edge if it joins two different components, checked with [Union-Find](/Software%20Engineering/DSA/02%20Data%20Structures/Union-Find). O(E log E). Natural with an edge list.
- **Prim**: grow one tree from a start vertex; always add the cheapest edge leaving the tree, using a min-heap. O(E log V). Natural with an adjacency list and dense graphs.

```ts
// Kruskal, using the DSU class from Union-Find
function kruskal(n: number, edges: [number, number, number][]): number {
  const dsu = new DSU(n);
  let total = 0, used = 0;
  for (const [u, v, w] of [...edges].sort((a, b) => a[2] - b[2])) {
    if (dsu.union(u, v)) {
      total += w;
      if (++used === n - 1) break;
    }
  }
  return used === n - 1 ? total : -1; // -1: graph not connected
}
```

```python
# Kruskal, using the DSU class from Union-Find
def kruskal(n: int, edges: list[tuple[int, int, int]]) -> int:
    dsu = DSU(n)
    total = used = 0
    for u, v, w in sorted(edges, key=lambda e: e[2]):
        if dsu.union(u, v):
            total += w
            used += 1
            if used == n - 1:
                break
    return total if used == n - 1 else -1  # -1: graph not connected
```

```ts
// Prim, using the Heap class from Heaps & Priority Queues. adj[u] = [[v, w], ...]
function prim(n: number, adj: [number, number][][]): number {
  const seen = new Array(n).fill(false);
  const pq = new Heap<[number, number]>((a, b) => a[0] < b[0]); // [edge weight, vertex]
  pq.push([0, 0]);
  let total = 0, count = 0;
  while (pq.size && count < n) {
    const [w, u] = pq.pop()!;
    if (seen[u]) continue;
    seen[u] = true;
    total += w;
    count++;
    for (const [v, wv] of adj[u]) if (!seen[v]) pq.push([wv, v]);
  }
  return count === n ? total : Infinity;
}
```

```python
import heapq

def prim(n: int, adj: list[list[tuple[int, int]]]) -> float:
    """adj[u] = [(v, w), ...], undirected. Returns MST weight or inf if disconnected."""
    seen = [False] * n
    pq = [(0, 0)]  # (edge weight, vertex)
    total, count = 0, 0
    while pq and count < n:
        w, u = heapq.heappop(pq)
        if seen[u]:
            continue
        seen[u] = True
        total += w
        count += 1
        for v, wv in adj[u]:
            if not seen[v]:
                heapq.heappush(pq, (wv, v))
    return total if count == n else float("inf")
```

## Complexity

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| Build adjacency list | O(V + E) | O(V + E) | |
| BFS / DFS | O(V + E) | O(V) | O(V^2) on a matrix |
| Topological sort (Kahn or DFS) | O(V + E) | O(V) | DAG only; detects cycles |
| Dijkstra (binary heap) | O((V + E) log V) | O(V + E) | Non-negative weights |
| Dijkstra (array, dense graph) | O(V^2) | O(V) | Better when E is close to V^2 |
| Bellman-Ford | O(V * E) | O(V) | Negative weights, detects negative cycles |
| Floyd-Warshall | O(V^3) | O(V^2) | All pairs |
| Kruskal | O(E log E) | O(V) | Dominated by sorting; DSU is near O(1) |
| Prim (binary heap) | O(E log V) | O(V + E) | |
| Connected components (DSU) | O(E * alpha(V)) | O(V) | See [Union-Find](/Software%20Engineering/DSA/02%20Data%20Structures/Union-Find) |

## Trade-offs & pitfalls

- **Forgetting `visited`** on a graph with cycles: infinite loop. Mark when you **enqueue** in BFS, not when you dequeue, or nodes get queued many times.
- **Edge direction**: prerequisites `[a, b]` often mean `b -> a`. Draw one example before coding.
- **Dijkstra with negative edges** gives wrong answers. Use Bellman-Ford (or SPFA, with care).
- **Dijkstra without the stale check** still works but does extra work; with a visited set, check it at pop time, not push time.
- **Disconnected graphs**: loop over every vertex as a possible start when counting components or detecting cycles.
- **Recursion depth** in DFS on large graphs (10^5 nodes in a line): use an explicit stack or raise the recursion limit in Python.
- **JS `Array.shift()`** is O(n); use a head index or a real deque.
- **Matrix on a sparse graph**: 10^5 vertices means 10^10 cells. Use lists.
- **Undirected cycle detection** with DFS must ignore the edge back to the parent; DSU is simpler.

## Senior perspective

- **Real systems are graphs**: package managers and bundlers resolve dependency DAGs (topological sort, cycle detection for circular imports), build tools (Bazel, Nx, Turborepo) schedule tasks by topo order and run independent ones in parallel ([Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos)), spreadsheets recompute cells in dependency order, garbage collectors do reachability from roots ([Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection)), routing uses Dijkstra and A* with heuristics and precomputation (contraction hierarchies), social feeds use BFS-like fan-out.
- **Scale changes the representation**: a billion-edge graph does not fit an in-memory adjacency list on one machine. You see compressed sparse row (CSR) arrays for cache-friendly traversal, graph databases (Neo4j), or batch systems (Pregel-style, GraphX). Many "graph" queries in practice are recursive SQL CTEs over an edges table.
- **Modeling is the hard part**: in interviews and in design, the skill is spotting that a problem is a graph (word ladder, bus routes, lock combinations, state machines) and choosing what a node and an edge are. After that the algorithm is usually standard.
- **What a senior asks in review**: can this dependency graph have cycles and what happens if it does? What is the max fan-out? Is recursion depth bounded? Are weights guaranteed non-negative? Is the traversal deterministic (sorted neighbors) when output order matters?

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 1971 | [Find if Path Exists in Graph](https://leetcode.com/problems/find-if-path-exists-in-graph/) | Easy | BFS/DFS or Union-Find |
| 200 | [Number of Islands](https://leetcode.com/problems/number-of-islands/) | Medium | Grid as implicit graph, flood fill |
| 133 | [Clone Graph](https://leetcode.com/problems/clone-graph/) | Medium | BFS/DFS with old-to-new map |
| 207 | [Course Schedule](https://leetcode.com/problems/course-schedule/) | Medium | Cycle detection via Kahn |
| 210 | [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) | Medium | Return the topological order |
| 743 | [Network Delay Time](https://leetcode.com/problems/network-delay-time/) | Medium | Dijkstra, answer is max distance |
| 787 | [Cheapest Flights Within K Stops](https://leetcode.com/problems/cheapest-flights-within-k-stops/) | Medium | Bellman-Ford limited to k + 1 rounds |
| 1584 | [Min Cost to Connect All Points](https://leetcode.com/problems/min-cost-to-connect-all-points/) | Medium | MST with Prim (dense) or Kruskal |

## Interview questions

<Accordion title="Adjacency list or matrix: which and why?">
List for sparse graphs (most real ones): O(V + E) space, neighbor iteration proportional to degree. Matrix for dense graphs or when you need O(1) edge lookups and V is small (a few thousand at most): O(V^2) space regardless of edges.
</Accordion>

<Accordion title="Why does BFS give shortest paths only on unweighted graphs?">
BFS visits vertices in order of number of edges from the source. With weights, a path with more edges can be cheaper, so you need Dijkstra (non-negative), 0-1 BFS (weights 0/1), or Bellman-Ford (negative).
</Accordion>

<Accordion title="Why does Dijkstra fail with negative edges?">
It assumes that once a vertex is popped with the smallest distance, no later path can improve it, because extending a path never makes it cheaper. A negative edge breaks that, so a settled vertex could later get a shorter distance. Bellman-Ford does not rely on that assumption.
</Accordion>

<Accordion title="How do you detect a cycle in a directed graph? In an undirected one?">
Directed: Kahn's algorithm (if not all vertices are output, there is a cycle) or DFS with three colors (an edge to a gray, in-progress node is a back edge). Undirected: DFS that ignores the parent edge, or Union-Find (an edge whose endpoints already share a root closes a cycle).
</Accordion>

<Accordion title="Kruskal vs Prim?">
Both give an MST. Kruskal sorts edges and uses Union-Find: simple, great for sparse graphs or edge lists, O(E log E). Prim grows from a vertex with a heap: good for dense graphs or adjacency lists, O(E log V), or O(V^2) with an array on complete graphs like LC 1584.
</Accordion>

<Accordion title="How would you model a word ladder or lock puzzle as a graph?">
Each state (word, lock combination) is a node; an edge connects states one move apart. Edges are generated on the fly (implicit graph). Since every move costs 1, BFS from the start state gives the minimum number of moves; bidirectional BFS cuts the search space further.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns) · [Tree Traversal - DFS & BFS](/Software%20Engineering/DSA/03%20Algorithms/Tree%20Traversal%20-%20DFS%20%26%20BFS) · [Union-Find](/Software%20Engineering/DSA/02%20Data%20Structures/Union-Find) · [Heaps & Priority Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Heaps%20%26%20Priority%20Queues)
- [Trees & Binary Search Trees](/Software%20Engineering/DSA/02%20Data%20Structures/Trees%20%26%20Binary%20Search%20Trees) · [Stacks & Queues](/Software%20Engineering/DSA/02%20Data%20Structures/Stacks%20%26%20Queues) · [Greedy](/Software%20Engineering/DSA/04%20Patterns/Greedy) (Dijkstra, MST) · [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming) (Bellman-Ford, Floyd-Warshall)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis) · [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection) · [Monorepos](/Software%20Engineering/Frontend/14%20Tooling/Monorepos)

## Sources
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022: ch. 20 (elementary graph algorithms), ch. 21 (MST), ch. 22 (single-source shortest paths)
- [Sedgewick & Wayne, Algorithms 4th ed.: Chapter 4 Graphs](https://algs4.cs.princeton.edu/40graphs/)
- [cp-algorithms: Dijkstra](https://cp-algorithms.com/graph/dijkstra.html)
- [cp-algorithms: Bellman-Ford](https://cp-algorithms.com/graph/bellman_ford.html)
- [cp-algorithms: Topological sorting](https://cp-algorithms.com/graph/topological-sort.html)
- [cp-algorithms: MST with Kruskal](https://cp-algorithms.com/graph/mst_kruskal.html) and [Prim](https://cp-algorithms.com/graph/mst_prim.html)
- [Tech Interview Handbook: Graph cheatsheet](https://www.techinterviewhandbook.org/algorithms/graph/)
- [NeetCode roadmap](https://neetcode.io/roadmap)
