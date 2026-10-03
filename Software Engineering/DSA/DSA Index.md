---
tags: [dsa, index]
aliases: [DSA Home, DSA MOC, Algorithms Home, Data Structures and Algorithms Index, LeetCode Home]
updated: 2026-09-27
---
# DSA Index

> [!tldr] Start here
> Entry point for Data Structures & Algorithms and coding-interview prep. Notes go from foundations (Big-O, how to approach a problem) through the core data structures and algorithms to the ~12 **patterns** that cover most interview questions. Every topic note has a complexity table, TypeScript and Python code, classic LeetCode problems and interview questions; every pattern note has a template, a worked example and practice problems.

```mermaid
mindmap
  root((DSA))
    Foundations
      Big-O
      Problem Solving
    Data Structures
      Arrays & Strings
      Linked Lists
      Stacks & Queues
      Hash Tables
      Trees & BSTs
      Heaps
      Graphs
      Tries
      Union-Find
    Algorithms
      Recursion
      Tree Traversal
      Sorting
      Dynamic Programming
    Patterns
      Two Pointers
      Sliding Window
      Prefix Sum
      Binary Search
      Fast & Slow
      Monotonic Stack
      Top-K
      Intervals
      Graph BFS/DFS
      Backtracking
      Greedy
      Bits
    Books & Courses
      Grokking Algorithms
      LeetCode Crash Course
```

## 🧭 Quick links
- 🎯 Interview mode: [[Problem Solving]] → the [[#Patterns → problems]] table → a curated list (below)
- 📐 Cheat sheet: [[Big-O & Complexity Analysis#Common complexities cheat sheet]]
- 📚 [[Grokking Algorithms]] (book) · [[LeetCode Crash Course]] (course map) · [[Data structure & Algorithms]] (old overview, maps the original sections to the new notes)

## 01 Foundations (beginner)
- [[Big-O & Complexity Analysis]]: time/space, amortized, what `n` you can afford, Master Theorem
- [[Problem Solving]]: the UMPIRE interview framework, clarifying questions, brute force → optimize, testing, AI-era interviews

## 02 Data Structures
| Beginner | Intermediate | Advanced |
|---|---|---|
| [[Arrays & Strings]] | [[Trees & Binary Search Trees]] | [[Graphs]] |
| [[Linked Lists]] | [[Heaps & Priority Queues]] | [[Tries]] |
| [[Stacks & Queues]] | | [[Union-Find]] |
| [[Hash Tables]] | | |

## 03 Algorithms
| Beginner | Intermediate | Advanced |
|---|---|---|
| [[Recursion & Backtracking]] | [[Tree Traversal - DFS & BFS]] | [[Dynamic Programming]] |
| [[Sorting Algorithms]] | | |

## 04 Patterns
| Beginner | Intermediate | Advanced |
|---|---|---|
| [[Two Pointers]] | [[Fast and Slow Pointer]] | [[Monotonic Stack & Queue]] |
| [[Sliding Window]] | [[Top-K & Heaps pattern]] | [[Backtracking]] |
| [[Prefix Sum]] | [[Intervals]] | [[Greedy]] |
| [[Binary Search]] | [[Graph BFS-DFS patterns]] | [[Bit Manipulation]] |

## 05 Books & Courses
- [[Grokking Algorithms]]: illustrated intro, chapter by chapter
- [[LeetCode Crash Course]]: LeetCode's Explore card, mapped to the notes here

## Patterns → problems
How to read it: spot the **signal** in the problem statement, reach for the **pattern**, practise with the **problems** (LeetCode numbers, easiest first).

| Pattern | Signals | Classic problems |
|---|---|---|
| [[Hash Tables\|Hashing]] | "seen before?", counts, pairs to a target, grouping | 1 Two Sum · 217 Contains Duplicate · 242 Valid Anagram · 49 Group Anagrams · 128 Longest Consecutive Sequence |
| [[Two Pointers]] | sorted array, pairs/triplets, palindromes, in-place partition | 125 Valid Palindrome · 167 Two Sum II · 15 3Sum · 11 Container With Most Water · 42 Trapping Rain Water |
| [[Sliding Window]] | longest/shortest **contiguous** subarray or substring with a condition | 643 Maximum Average Subarray I · 209 Minimum Size Subarray Sum · 3 Longest Substring Without Repeating Characters · 424 Longest Repeating Character Replacement · 76 Minimum Window Substring |
| [[Prefix Sum]] | range sums, "subarray sum equals k", products except self | 303 Range Sum Query · 238 Product of Array Except Self · 560 Subarray Sum Equals K · 525 Contiguous Array · 304 Range Sum Query 2D |
| [[Binary Search]] | sorted input, "minimum X such that...", O(log n) required | 704 Binary Search · 35 Search Insert Position · 153 Find Minimum in Rotated Sorted Array · 33 Search in Rotated Sorted Array · 875 Koko Eating Bananas · 4 Median of Two Sorted Arrays |
| [[Fast and Slow Pointer]] | linked list cycle, middle, "number sequence loops" | 876 Middle of the Linked List · 141 Linked List Cycle · 202 Happy Number · 142 Linked List Cycle II · 287 Find the Duplicate Number |
| [[Linked Lists\|Linked list manipulation]] | reverse, merge, reorder, dummy head | 206 Reverse Linked List · 21 Merge Two Sorted Lists · 19 Remove Nth Node From End · 143 Reorder List · 146 LRU Cache |
| [[Monotonic Stack & Queue]] | next greater/smaller, spans, histogram, window max | 20 Valid Parentheses (plain stack) · 496 Next Greater Element I · 739 Daily Temperatures · 84 Largest Rectangle in Histogram · 239 Sliding Window Maximum |
| [[Top-K & Heaps pattern]] | k largest/smallest/most frequent, merge k sorted, running median | 215 Kth Largest Element · 973 K Closest Points to Origin · 347 Top K Frequent Elements · 23 Merge k Sorted Lists · 295 Find Median from Data Stream |
| [[Intervals]] | ranges, meetings, overlaps, calendars | 56 Merge Intervals · 57 Insert Interval · 435 Non-overlapping Intervals · 986 Interval List Intersections · 253 Meeting Rooms II (premium) |
| [[Tree Traversal - DFS & BFS\|Trees]] | hierarchy, depth, paths, levels, BST ordering | 104 Maximum Depth · 226 Invert Binary Tree · 102 Level Order Traversal · 98 Validate BST · 236 Lowest Common Ancestor · 124 Maximum Path Sum · 297 Serialize and Deserialize |
| [[Graph BFS-DFS patterns]] | grids, islands, shortest steps, dependencies, "clone" | 200 Number of Islands · 133 Clone Graph · 994 Rotting Oranges · 207 Course Schedule · 417 Pacific Atlantic Water Flow · 127 Word Ladder |
| [[Union-Find]] | connectivity, merging groups, redundant edge | 547 Number of Provinces · 684 Redundant Connection · 721 Accounts Merge |
| [[Tries]] | prefixes, autocomplete, many words on a board | 208 Implement Trie · 211 Add and Search Words · 212 Word Search II |
| [[Backtracking]] | "all" subsets/permutations/combinations, constraint puzzles | 78 Subsets · 46 Permutations · 39 Combination Sum · 17 Letter Combinations · 79 Word Search · 51 N-Queens |
| [[Greedy]] | local best choice provably works, sort by end/deadline | 55 Jump Game · 763 Partition Labels · 134 Gas Station · 45 Jump Game II · 621 Task Scheduler |
| [[Dynamic Programming]] | count ways / min cost / max value with overlapping subproblems | 70 Climbing Stairs · 198 House Robber · 322 Coin Change · 300 Longest Increasing Subsequence · 1143 Longest Common Subsequence · 416 Partition Equal Subset Sum · 72 Edit Distance |
| [[Bit Manipulation]] | "no extra space", XOR pairs, powers of two, subsets as masks | 136 Single Number · 191 Number of 1 Bits · 338 Counting Bits · 268 Missing Number · 371 Sum of Two Integers |

Links to each problem are in the pattern notes.

## Curated problem lists
| List | Size | Best for |
|---|---|---|
| **Blind 75** | 75 | The classic minimum set, if time is short |
| **Grind 75** | 75 by default, configurable (up to 169) | A generated week-by-week schedule for the hours you have, by the Blind 75 author |
| **NeetCode 150** | 150 | Blind 75 plus 75 more, grouped by pattern, with video solutions; best for learning patterns |

Pick **one**, finish it, then redo the ones you got wrong. More problems don't help as much as understanding each pattern.

## Suggested study path
> [!example]- 🌱 Beginner (weeks 1–3): foundations and linear structures
> 1. [[Big-O & Complexity Analysis]] → [[Problem Solving]]
> 2. [[Arrays & Strings]] → [[Hash Tables]] → [[Two Pointers]] → [[Sliding Window]] → [[Prefix Sum]]
> 3. [[Linked Lists]] → [[Fast and Slow Pointer]] → [[Stacks & Queues]]
> 4. [[Binary Search]] · [[Sorting Algorithms]] · [[Recursion & Backtracking]]
> 5. Optional companion: [[Grokking Algorithms]] chapters 1–5, or the first chapters of [[LeetCode Crash Course]]
> Goal: solve Easy problems in ~15 minutes and explain their Big-O.

> [!example]- 🌿 Intermediate (weeks 4–7): trees, graphs, heaps
> 1. [[Trees & Binary Search Trees]] → [[Tree Traversal - DFS & BFS]]
> 2. [[Heaps & Priority Queues]] → [[Top-K & Heaps pattern]] → [[Intervals]]
> 3. [[Graphs]] → [[Graph BFS-DFS patterns]] → [[Union-Find]]
> 4. [[Monotonic Stack & Queue]] · [[Tries]]
> Goal: solve most Mediums in 25–30 minutes using the [[Problem Solving]] framework out loud.

> [!example]- 🌳 Senior interview level (weeks 8–12): search, optimization, polish
> 1. [[Backtracking]] → [[Dynamic Programming]] (1D → 2D → knapsack → strings) → [[Greedy]] → [[Bit Manipulation]]
> 2. Advanced graphs in [[Graphs]]: topological sort, Dijkstra, MST
> 3. Work through one curated list (NeetCode 150 or Grind 75) under timed conditions; do mock interviews out loud
> 4. Practise follow-ups a senior is expected to handle: streaming input, memory limits, concurrency, "how would this run across machines", and reading/debugging unfamiliar code (AI-assisted rounds)
> Goal: optimal solution plus clean code, tests and trade-off discussion within 35–40 minutes, and handle Hards with hints.

> [!example]- ⏱️ One week before an interview
> [[Problem Solving]] (reread) · the [[#Patterns → problems]] table, one pattern per session, 2–3 problems each · [[Big-O & Complexity Analysis#Common complexities cheat sheet]] · ask the recruiter whether AI tools are allowed and in which rounds.

## 🆕 2025–2026 in this area
- **AI-enabled coding interviews**: Meta piloted a coding round with a built-in AI assistant from October 2025 (bug fixing, implementation, optimization in a multi-file codebase); Canva replaced its CS fundamentals screen with an AI-assisted coding interview (June 2025). → [[Problem Solving]]
- **Companies restricting AI**: Amazon warned candidates they can be disqualified for unauthorized GenAI use (2025); Anthropic allows Claude for application materials but not in live interviews unless told; Google is reintroducing at least one in-person round. → [[Problem Solving]]
- **What changed for prep**: more weight on reading, debugging and verifying code, less on reproducing memorized solutions. → [[Problem Solving]]
- **Python 3.14** (2025-10-07) adds public max-heap functions to `heapq` (`heappush_max`, `heappop_max`, ...), no more negation trick. → [[Heaps & Priority Queues]] · [[Top-K & Heaps pattern]]
- **ES2025** adds `Set` methods (`union`, `intersection`, `difference`, ...) to JavaScript. → [[Hash Tables]]

## Related
- [[Software Engineering]], the vault root
- [[Frontend Home]] · [[Frontend System Design Index]] (system design rounds)
- `_Archive` holds the originals replaced in this restructure: the empty `Data structure & Algorithms` skeleton, `LC - Crash Course` (now [[Sliding Window]]) and `Arrays and strings` (now [[Two Pointers]])

## Sources
- [LeetCode Explore: Interview Crash Course, DSA](https://leetcode.com/explore/interview/card/leetcodes-interview-crash-course-data-structures-and-algorithms/)
- [Tech Interview Handbook: Grind 75](https://www.techinterviewhandbook.org/grind75) · [Algorithms study cheatsheet](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/)
- [NeetCode 150](https://neetcode.io/practice/practice/neetcode150) · [Blind 75](https://neetcode.io/practice/practice/blind75) · [NeetCode roadmap](https://neetcode.io/roadmap)
- [Hello Interview: Meta's AI-Enabled Coding Interview](https://www.hellointerview.com/blog/meta-ai-enabled-coding) · [404 Media (2025)](https://www.404media.co/meta-is-going-to-let-job-candidates-use-ai-during-coding-tests/)
- [Canva Engineering (2025-06-11)](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/)
- [Python 3.14 What's New](https://docs.python.org/3/whatsnew/3.14.html)
- [TC39 finished proposals](https://github.com/tc39/proposals/blob/main/finished-proposals.md)
