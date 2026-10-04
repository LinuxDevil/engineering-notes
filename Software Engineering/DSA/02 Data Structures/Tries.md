---
title: "Tries"
tags: [dsa, dsa/data-structures]
aliases: [Trie, Prefix Tree, Digital Tree, Radix Tree, Radix Trie, Patricia Trie, Compressed Trie]
updated: 2026-09-27
---
# Tries

<Note>
**TL;DR**

A **trie** (prefix tree) stores strings character by character along paths from the root, so all words sharing a prefix share the same nodes. Insert, search and `startsWith` cost **O(L)** where L is the word length, **independent of how many words are stored**. It shines for **autocomplete, prefix queries and searching many words at once** (Word Search II). The price is memory: one node per character; **compressed (radix) tries** merge single-child chains to cut it down.
</Note>

## The idea

Store the words `car`, `cat`, `cart`, `dog`:

{/* diagram source: /diagrams/diagram-192.mmd */}
![diagram](/diagrams/diagram-192.svg)

- Each **edge** is a character. Each **node** is a prefix (the path from the root).
- An **end-of-word flag** marks nodes where a full word finishes. Without it, you couldn't tell that `car` is a word but `ca` is not.
- To check "does any word start with `ca`?", walk `c`, `a`. If the path exists, the answer is yes. The number of stored words doesn't matter.

Compare with a hash set of words: exact lookup is also O(L) (you hash the whole string), but a hash set **cannot answer prefix questions** without scanning every key.

The name comes from re**trie**val (Fredkin, 1960). People say "try" to distinguish it from "tree".

## How it works

| Operation | Steps |
|---|---|
| `insert(word)` | from root, for each char: go to the child, creating it if missing. Mark the last node `isEnd = true`. |
| `search(word)` | walk each char; if a child is missing, return false. At the end, return `node.isEnd`. |
| `startsWith(prefix)` | walk each char; if all exist, return true. |
| `delete(word)` | unmark `isEnd`; then, bottom-up, remove nodes that have no children and aren't the end of another word. |
| autocomplete(prefix) | walk to the prefix node, then DFS below it to collect words (optionally keep top-k per node). |

### Child storage options

| Children as | Memory | Speed | When |
|---|---|---|---|
| Fixed array of 26 (or 256) | 26 pointers per node, mostly null | fastest, index by `c - 'a'` | small known alphabet, interview "lowercase letters" |
| Hash map (`Map` / `dict`) | only existing edges | fast, some hashing overhead | large or unknown alphabet (Unicode) |
| Sorted array / linked list | smallest | O(alphabet) per step | memory-critical |

### Compressed (radix / Patricia) tries

A plain trie for `interview`, `internal`, `internet` has long chains of single-child nodes. A **radix trie** stores a **substring on each edge** and merges any node that has one child and isn't a word end.

```text
Plain trie                       Radix trie
i-n-t-e-r-v-i-e-w                root
          \                        └─ "inter"
           n-a-l                        ├─ "view"  (end)
            \                           └─ "n"
             e-t                            ├─ "al" (end)
                                            └─ "et" (end)
```

The number of nodes drops from O(total characters) to O(number of words). Inserting may **split** an edge when a new word diverges mid-edge. Patricia tries (Morrison, 1968) are the binary version used in IP routing.

## Complexity

L = length of the word/prefix, N = number of words, Σ = alphabet size, M = total characters inserted.

| Operation | Time | Space |
|---|---|---|
| Insert | O(L) | O(L) new nodes worst case |
| Search / startsWith | O(L) | O(1) |
| Delete | O(L) | O(1) |
| Autocomplete (collect all under a prefix) | O(L + size of subtree) | O(output) |
| Autocomplete top-k with cached suggestions per node | O(L + k) | O(M * k) extra |
| Build from N words | O(M) | O(M) nodes; O(M * Σ) pointers with array children |
| Radix trie nodes | | O(N) nodes, edges hold substrings |

## Implementation

### Trie with map children (LeetCode 208)

```ts
class TrieNode {
  children = new Map<string, TrieNode>();
  isEnd = false;
}

class Trie {
  private root = new TrieNode();

  insert(word: string): void {
    let node = this.root;
    for (const ch of word) {
      let next = node.children.get(ch);
      if (!next) {
        next = new TrieNode();
        node.children.set(ch, next);
      }
      node = next;
    }
    node.isEnd = true;
  }

  private walk(s: string): TrieNode | null {
    let node = this.root;
    for (const ch of s) {
      const next = node.children.get(ch);
      if (!next) return null;
      node = next;
    }
    return node;
  }

  search(word: string): boolean {
    return this.walk(word)?.isEnd ?? false;
  }

  startsWith(prefix: string): boolean {
    return this.walk(prefix) !== null;
  }

  // Collect up to `limit` words starting with prefix, in lexicographic order.
  autocomplete(prefix: string, limit = 10): string[] {
    const start = this.walk(prefix);
    const out: string[] = [];
    const dfs = (node: TrieNode, path: string): void => {
      if (out.length >= limit) return;
      if (node.isEnd) out.push(path);
      const keys = [...node.children.keys()].sort();
      for (const ch of keys) dfs(node.children.get(ch)!, path + ch);
    };
    if (start) dfs(start, prefix);
    return out;
  }

  delete(word: string): boolean {
    // returns true if the child node can be pruned
    const remove = (node: TrieNode, i: number): boolean => {
      if (i === word.length) {
        if (!node.isEnd) return false;
        node.isEnd = false;
        return node.children.size === 0;
      }
      const child = node.children.get(word[i]);
      if (!child) return false;
      if (remove(child, i + 1)) node.children.delete(word[i]);
      return !node.isEnd && node.children.size === 0;
    };
    const existed = this.search(word);
    if (existed) remove(this.root, 0);
    return existed;
  }
}
```

```python
class TrieNode:
    __slots__ = ("children", "is_end")

    def __init__(self):
        self.children: dict[str, "TrieNode"] = {}
        self.is_end = False


class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def _walk(self, s: str) -> TrieNode | None:
        node = self.root
        for ch in s:
            node = node.children.get(ch)
            if node is None:
                return None
        return node

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_end

    def startsWith(self, prefix: str) -> bool:
        return self._walk(prefix) is not None

    def autocomplete(self, prefix: str, limit: int = 10) -> list[str]:
        start = self._walk(prefix)
        out: list[str] = []
        if start is None:
            return out
        stack = [(start, prefix)]              # iterative DFS, no recursion limit
        while stack and len(out) < limit:
            node, path = stack.pop()
            if node.is_end:
                out.append(path)
            # push in reverse so smaller letters are popped first
            for ch in sorted(node.children, reverse=True):
                stack.append((node.children[ch], path + ch))
        return out
```

`setdefault(ch, TrieNode())` builds a throwaway node even when the child exists. It's fine for clarity; in hot code write `child = node.children.get(ch)` then create only if missing.

### Array children (fast, lowercase only)

```ts
class ArrayTrieNode {
  children: (ArrayTrieNode | null)[] = new Array(26).fill(null);
  isEnd = false;
}

function insertArr(root: ArrayTrieNode, word: string): void {
  let node = root;
  for (let i = 0; i < word.length; i++) {
    const idx = word.charCodeAt(i) - 97; // 'a'
    node = node.children[idx] ??= new ArrayTrieNode();
  }
  node.isEnd = true;
}
```

```python
class ArrayTrieNode:
    __slots__ = ("children", "is_end")

    def __init__(self):
        self.children: list["ArrayTrieNode | None"] = [None] * 26
        self.is_end = False


def insert_arr(root: ArrayTrieNode, word: str) -> None:
    node = root
    for ch in word:
        i = ord(ch) - ord("a")
        if node.children[i] is None:
            node.children[i] = ArrayTrieNode()
        node = node.children[i]
    node.is_end = True
```

### Wildcard search (LeetCode 211)

`.` matches any letter, so at a dot you branch into every child (DFS).

```ts
function searchWithDots(node: TrieNode, word: string, i = 0): boolean {
  if (i === word.length) return node.isEnd;
  const ch = word[i];
  if (ch === ".") {
    for (const child of node.children.values()) {
      if (searchWithDots(child, word, i + 1)) return true;
    }
    return false;
  }
  const next = node.children.get(ch);
  return next ? searchWithDots(next, word, i + 1) : false;
}
```

```python
def search_with_dots(node: TrieNode, word: str, i: int = 0) -> bool:
    if i == len(word):
        return node.is_end
    ch = word[i]
    if ch == ".":
        return any(search_with_dots(child, word, i + 1) for child in node.children.values())
    nxt = node.children.get(ch)
    return nxt is not None and search_with_dots(nxt, word, i + 1)
```

### Word Search II (LeetCode 212): search many words on a grid at once

Brute force runs a grid DFS per word. Instead, put all words in a trie and run **one** DFS from each cell that follows trie edges. The trie prunes any path that isn't a prefix of some word.

Key optimizations:
- Store the full word at its end node (`word` field) so you don't rebuild strings.
- After finding a word, clear it (`word = null`) to avoid duplicates.
- **Prune** leaf nodes after use so later searches skip exhausted branches.
- Mark visited cells in place (`'#'`) and restore on backtrack ([Backtracking](/Software%20Engineering/DSA/04%20Patterns/Backtracking)).

```ts
function findWords(board: string[][], words: string[]): string[] {
  type Node = { children: Map<string, Node>; word: string | null };
  const root: Node = { children: new Map(), word: null };
  for (const w of words) {
    let node = root;
    for (const ch of w) {
      if (!node.children.has(ch)) node.children.set(ch, { children: new Map(), word: null });
      node = node.children.get(ch)!;
    }
    node.word = w;
  }

  const rows = board.length, cols = board[0].length;
  const found: string[] = [];

  const dfs = (r: number, c: number, parent: Node): void => {
    const ch = board[r][c];
    const node = parent.children.get(ch);
    if (!node) return;
    if (node.word !== null) { found.push(node.word); node.word = null; }

    board[r][c] = "#";
    for (const [dr, dc] of [[1, 0], [-1, 0], [0, 1], [0, -1]]) {
      const nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && board[nr][nc] !== "#") dfs(nr, nc, node);
    }
    board[r][c] = ch;

    if (node.children.size === 0 && node.word === null) parent.children.delete(ch); // prune
  };

  for (let r = 0; r < rows; r++) for (let c = 0; c < cols; c++) dfs(r, c, root);
  return found;
}
```

```python
def find_words(board: list[list[str]], words: list[str]) -> list[str]:
    root: dict = {}
    END = "$"                                   # key holding the full word
    for w in words:
        node = root
        for ch in w:
            node = node.setdefault(ch, {})
        node[END] = w

    rows, cols = len(board), len(board[0])
    found: list[str] = []

    def dfs(r: int, c: int, parent: dict) -> None:
        ch = board[r][c]
        node = parent.get(ch)
        if node is None:
            return
        word = node.pop(END, None)
        if word is not None:
            found.append(word)

        board[r][c] = "#"
        for nr, nc in ((r + 1, c), (r - 1, c), (r, c + 1), (r, c - 1)):
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] in node:
                dfs(nr, nc, node)
        board[r][c] = ch

        if not node:                            # prune exhausted branch
            del parent[ch]

    for r in range(rows):
        for c in range(cols):
            if board[r][c] in root:
                dfs(r, c, root)
    return found
```

The Python version uses nested dicts as nodes, a common compact trie idiom. `'#'` is never a trie key (words are letters), so `board[nr][nc] in node` also skips visited cells.

## Trade-offs & pitfalls

- **Forgetting `isEnd`**: `search("ca")` returns true after inserting `car` if you only check that the path exists. That's `startsWith`, not `search`.
- **Memory blow-up**: 26-slot arrays for 10^5 words with 10 chars each is ~10^6 nodes x 26 pointers. Use maps, a radix trie, or a single flat array of nodes (`children[node * 26 + c]`).
- **Deleting incorrectly**: removing nodes that are prefixes of other words (`car` when `cart` exists) or ends of other words. Only prune nodes with no children and no `isEnd`.
- **Recursion depth**: recursive DFS over a trie is bounded by word length, usually fine. Autocomplete over huge subtrees should be iterative or limited to top-k.
- **String building in DFS**: `path + ch` creates a new string each step. For long words, keep a char array (push/pop) and `join` at word ends.
- **Unicode**: in JS, `for...of` iterates code points, while `word[i]` and `charCodeAt` are UTF-16 units. Be consistent. Normalize (`str.normalize("NFC")`) before inserting user text.
- **When a hash set is enough**: if you only need exact lookups, a `Set` is simpler and usually faster. Use a trie when prefixes matter.

## Senior perspective

- **Real-world uses**:
  - **Autocomplete / typeahead**: tries with cached top-k suggestions per node, ranked by frequency. At scale, this is often a precomputed prefix-to-suggestions map sharded across servers, rebuilt offline, with the trie used to build it.
  - **IP routing (longest-prefix match)**: routers use binary / Patricia / multibit tries over address bits.
  - **HTTP routers**: many web frameworks match URL paths with radix trees (for example Go's `httprouter`, and routers in Fastify via `find-my-way`).
  - **Spell checkers, word games, T9**; **DNA k-mer indexes**.
  - **Ethereum's Merkle Patricia Trie** stores account state as a hashed radix trie.
  - **Full-text search engines** (Lucene) store their term dictionary in **FSTs** (finite state transducers), which share both prefixes and suffixes, far more compact than a trie.
- **Related structures**: **suffix tries / suffix trees / suffix arrays** answer substring queries; **Aho-Corasick** adds failure links to a trie to find many patterns in a text in one pass (used in grep-like tools and malware signature scanning); a **bitwise trie** over integers solves maximum XOR queries.
- **Memory and locality**: pointer-heavy tries have poor cache behaviour. Production versions use arrays of nodes with integer indexes, double-array tries, or succinct/FST encodings.
- **In review, ask**: how many keys and how long? Is the alphabet bounded? Is it read-mostly (then build once, compress, maybe serialize) or updated live (then plain or radix trie)? Is a sorted array with [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search) on prefixes enough (`bisect` to the first key >= prefix, scan while it matches)? That's often simpler and smaller.
- **In interviews**: the signal for a trie is "many words, prefix queries" or "search a set of words simultaneously". State the O(L) bounds and the memory trade-off, and mention the radix trie as the space optimization.

## Classic LeetCode problems

| # | Problem | Difficulty | Key idea |
|---|---|---|---|
| 208 | [Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | Medium | insert / search / startsWith with end flag |
| 211 | [Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/) | Medium | DFS into all children on `.` |
| 212 | [Word Search II](https://leetcode.com/problems/word-search-ii/) | Hard | trie of words + grid backtracking + pruning |
| 1268 | [Search Suggestions System](https://leetcode.com/problems/search-suggestions-system/) | Medium | trie with top-3 per node, or sort + binary search |
| 648 | [Replace Words](https://leetcode.com/problems/replace-words/) | Medium | walk each word until the first root end |
| 720 | [Longest Word in Dictionary](https://leetcode.com/problems/longest-word-in-dictionary/) | Medium | DFS only through end-of-word nodes |
| 421 | [Maximum XOR of Two Numbers in an Array](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/) | Medium | bitwise trie, greedily take the opposite bit |

## Interview questions

<Accordion title="What's the time complexity of trie operations and why is it independent of the number of words?">
Insert, search and startsWith are O(L) for a string of length L: each step follows one edge. The number of stored words affects memory, not the path length.
</Accordion>

<Accordion title="Trie vs hash set: when would you choose each?">
Hash set for exact membership: simpler, less memory, O(L) average to hash. Trie when you need prefix queries, autocomplete, lexicographic enumeration, or searching many words at once (grid, streaming text).
</Accordion>

<Accordion title="Why do you need an end-of-word marker?">
Because a node exists for every prefix. Without the flag, inserting `apple` would make `app` look like a stored word.
</Accordion>

<Accordion title="How do you reduce a trie's memory use?">
Use hash-map children instead of fixed arrays, compress single-child chains into a radix (Patricia) trie, store nodes in flat arrays with integer indexes, or use a DAWG/FST that also shares suffixes.
</Accordion>

<Accordion title="Explain the trie approach to Word Search II.">
Build a trie of all words. DFS from every cell, following the trie in lockstep with the grid path; stop as soon as the current path is not a prefix. Record words at end nodes, clear them to avoid duplicates, and prune empty branches. One DFS serves every word.
</Accordion>

<Accordion title="How would you design autocomplete for a search box?">
Trie keyed by normalized query prefixes, each node caching the top-k completions by frequency so a lookup is O(L + k). Rebuild offline from logs, shard by prefix, cache hot prefixes, debounce requests on the client. See [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index) for the UI side.
</Accordion>

<Accordion title="How do you delete a word from a trie?">
Walk to its end node and unset `isEnd`. Then, unwinding the recursion, remove each child node that has no children and is not the end of another word. Stop at the first node still in use.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index)
- [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis) · [Trees & Binary Search Trees](/Software%20Engineering/DSA/02%20Data%20Structures/Trees%20%26%20Binary%20Search%20Trees) · [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) · [Arrays & Strings](/Software%20Engineering/DSA/02%20Data%20Structures/Arrays%20%26%20Strings) · [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs)
- Algorithms and patterns: [Recursion & Backtracking](/Software%20Engineering/DSA/03%20Algorithms/Recursion%20%26%20Backtracking) · [Backtracking](/Software%20Engineering/DSA/04%20Patterns/Backtracking) · [Tree Traversal - DFS & BFS](/Software%20Engineering/DSA/03%20Algorithms/Tree%20Traversal%20-%20DFS%20%26%20BFS) · [Bit Manipulation](/Software%20Engineering/DSA/04%20Patterns/Bit%20Manipulation) · [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search)
- [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course)

## Sources
- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press 2022 (radix trees, problem 12-2)
- [Sedgewick & Wayne, Algorithms 4th ed.: Tries (5.2)](https://algs4.cs.princeton.edu/52trie/)
- [cp-algorithms: Aho-Corasick algorithm (trie construction)](https://cp-algorithms.com/string/aho_corasick.html)
- Fredkin, E., "Trie Memory", *Communications of the ACM* 3(9), 1960 · Morrison, D. R., "PATRICIA", *Journal of the ACM* 15(4), 1968
- [Ethereum docs: Merkle Patricia Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)
- [julienschmidt/httprouter (radix tree router)](https://github.com/julienschmidt/httprouter)
- [Apache Lucene FST package docs](https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/util/fst/package-summary.html)
- [LeetCode Explore: Trie card](https://leetcode.com/explore/learn/card/trie/)
- [NeetCode roadmap](https://neetcode.io/roadmap)
