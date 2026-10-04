---
title: "Problem Solving"
tags: [dsa, dsa/foundations, interview]
aliases: [Problem Solving Framework, UMPIRE, Coding Interview Framework, How to Approach a Coding Problem, Coding Interview Strategy]
updated: 2026-09-27
---
# Problem Solving

<Note>
**TL;DR**

A coding interview grades **how you think**, not just whether the code passes. Use a repeatable framework such as **UMPIRE** (Understand, Match, Plan, Implement, Review, Evaluate): clarify the problem and edge cases, match it to a known [pattern](/Software%20Engineering/DSA/DSA%20Index#patterns-problems), state a brute force, optimize it, code cleanly while talking, then test by hand and give the complexity. Since 2025 some companies (Meta, Canva) run AI-assisted rounds that add debugging and verifying AI output to the same core skills.
</Note>

## The idea
Most interview problems are variations of ~15 patterns. The hard part isn't knowing the patterns, it's reliably getting from a vague prompt to a correct, explained solution under time pressure. A framework removes the "blank page" moment and makes sure you hit every signal the interviewer is scoring:

| Signal | What they look for |
|---|---|
| Communication | You think out loud, explain trade-offs, respond to hints |
| Problem solving | You clarify, find a working approach, then improve it |
| Technical competence | Correct, idiomatic, readable code, reasonably fast |
| Testing | You catch your own bugs and edge cases before the interviewer does |

## UMPIRE, step by step

{/* diagram source: /diagrams/diagram-185.mmd */}
![diagram](/diagrams/diagram-185.svg)

Rough time budget for a 45-minute round: 5 min understand, 5–10 min match and plan, 15–20 min implement, 5 min review, 2–3 min evaluate, rest for follow-ups.

### 1. Understand (clarify before you code)
Restate the problem in your own words, then ask. Good clarifying questions:

- **Input**: size range (`n ≤ 10⁵`?), types, can it be empty? negative numbers? duplicates? sorted? Unicode or ASCII?
- **Output**: return indices or values? any valid answer or all of them? what if there is no answer (`-1`, `null`, throw)?
- **Constraints**: can I mutate the input? memory limits? is it a stream (can't see all data at once)?
- **Edge cases**: empty, one element, all equal, very large values (overflow), cycles in graphs, disconnected components.

Write 2–3 examples yourself, including one edge case. Walking an example by hand often reveals the algorithm.

### 2. Match (recognize the pattern)
Use the constraints and keywords as signals:

| Signal in the problem | Try |
|---|---|
| Sorted array, "find pair/triplet" | [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers), [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search) |
| Contiguous subarray/substring, "longest/shortest with condition" | [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window) |
| "Sum of subarray", range queries | [Prefix Sum](/Software%20Engineering/DSA/04%20Patterns/Prefix%20Sum) (+ hash map) |
| "Have I seen this before?", counts, pairs summing to target | [Hash Tables](/Software%20Engineering/DSA/02%20Data%20Structures/Hash%20Tables) |
| Linked list cycle or middle | [Fast and Slow Pointer](/Software%20Engineering/DSA/04%20Patterns/Fast%20and%20Slow%20Pointer) |
| "Next greater/smaller", histogram | [Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) |
| "k largest/smallest/most frequent", streaming median | [Top-K & Heaps pattern](/Software%20Engineering/DSA/04%20Patterns/Top-K%20%26%20Heaps%20pattern) |
| Overlapping ranges, meetings, calendars | [Intervals](/Software%20Engineering/DSA/04%20Patterns/Intervals) |
| Grid, islands, shortest steps, dependencies | [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns), [Graphs](/Software%20Engineering/DSA/02%20Data%20Structures/Graphs) |
| Tree questions | [Tree Traversal - DFS & BFS](/Software%20Engineering/DSA/03%20Algorithms/Tree%20Traversal%20-%20DFS%20%26%20BFS) |
| "All combinations/permutations/subsets" | [Backtracking](/Software%20Engineering/DSA/04%20Patterns/Backtracking) |
| "Min/max ways/cost", choices with overlapping subproblems | [Dynamic Programming](/Software%20Engineering/DSA/03%20Algorithms/Dynamic%20Programming) |
| Locally best choice seems to work, intervals by end | [Greedy](/Software%20Engineering/DSA/04%20Patterns/Greedy) |
| Prefix lookups, autocomplete, word dictionary | [Tries](/Software%20Engineering/DSA/02%20Data%20Structures/Tries) |
| Connectivity, merging groups | [Union-Find](/Software%20Engineering/DSA/02%20Data%20Structures/Union-Find) |
| "Without extra space", XOR, powers of two | [Bit Manipulation](/Software%20Engineering/DSA/04%20Patterns/Bit%20Manipulation) |
| Monotonic answer ("minimum capacity such that...") | [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search) on the answer |

Use `n` to guess the target complexity (see [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis#what-input-size-can-i-afford)).

### 3. Plan (brute force → optimize)
1. **State the brute force** and its complexity out loud, even if it's obviously slow. It proves you understand the problem and gives you a fallback.
2. **Find the bottleneck**: which part is repeated work?
3. **Optimize with a standard move**:
   - repeated lookups → hash map/set
   - repeated range sums → prefix sums
   - sorted input → two pointers / binary search
   - repeated subproblems → memoization
   - need the min/max repeatedly → heap
   - nested loops over a contiguous range → sliding window
4. **Agree on the approach** with the interviewer before coding. Write a short pseudocode outline as comments.

### 4. Implement
- Use clear names (`left`, `right`, `windowCount`), not `a`, `b`, `tmp2`.
- Write small helpers (`isValid`, `neighbors`) rather than one huge function.
- Keep talking: say what each block does and why.
- If you get stuck on a detail, leave a `// TODO: handle X` and keep moving; come back after the main flow works.
- Use the language you're fastest in and know its standard library (Python `collections`, `heapq`, `bisect`; TS `Map`, `Set`, and a small heap class).

### 5. Review (test like a senior)
- **Trace your code** line by line on a small example, tracking variables in a table. Don't just re-read it.
- Then run the edge cases: empty, single element, duplicates, negatives, max values.
- Common bugs to check: off-by-one in loop bounds, `<` vs `<=` in binary search, forgetting to update a pointer (infinite loop), mutating a list while iterating, not resetting state between test cases, integer overflow in `mid = (lo + hi) / 2` (in Java/C++).

### 6. Evaluate
- Give **time and space complexity** and justify them.
- Mention trade-offs and alternatives ("if memory were tight I'd sort in place instead of using a set, O(n log n) time, O(1) space").
- Discuss follow-ups: streaming input, distributed data, very large `n`, concurrency.

## Worked example: Two Sum (LC 1)

> Given `nums` and `target`, return indices of two numbers that add up to `target`.

1. **Understand**: exactly one solution? (yes) can I use the same element twice? (no) return order? (any). Example: `[2,7,11,15], 9 → [0,1]`.
2. **Match**: "pair that sums to target" + unsorted → hash map (or sort + [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers), but that loses indices).
3. **Plan**: brute force is all pairs, O(n²). Bottleneck: for each `x` we search for `target - x` linearly. Use a map `value → index` to make that lookup O(1).
4. **Implement**:

```ts
function twoSum(nums: number[], target: number): [number, number] {
  const indexOf = new Map<number, number>();      // value -> index
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    const j = indexOf.get(need);
    if (j !== undefined) return [j, i];
    indexOf.set(nums[i], i);                      // insert after the check: no reuse of i
  }
  throw new Error("no solution");
}
```

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    index_of: dict[int, int] = {}
    for i, x in enumerate(nums):
        if target - x in index_of:
            return [index_of[target - x], i]
        index_of[x] = i
    raise ValueError("no solution")
```

5. **Review**: trace `[3,3], 6`: i=0 map empty, store 3→0; i=1 need 3, found 0 → `[0,1]`. Duplicates work because we check before inserting.
6. **Evaluate**: O(n) time, O(n) space. Trade-off vs sort + two pointers: O(n log n) time, O(1) extra space if we don't need original indices.

## Communicating well
- **Think out loud**, but in sentences, not a stream of fragments. Silence for 3+ minutes is the most common failure mode.
- **Take hints gracefully**: a hint is not a penalty, ignoring it is.
- **Admit uncertainty**: "I think `heapq` is a min-heap by default, so I'll negate the values" is better than guessing silently.
- **Don't jump to code**. Interviewers explicitly score whether you clarified first.
- **Keep the interviewer in the loop** when you change approach: "the window idea fails because values can be negative, switching to prefix sums with a hash map".

## Practising effectively
- Learn **patterns, not problems**. After each problem write one line: "signal → pattern → key trick".
- Use a curated list rather than random problems:
  - **Blind 75**: the original 75-problem list.
  - **Grind 75**: by the Blind 75 author; generates a week-by-week schedule for the time you have (Tech Interview Handbook).
  - **NeetCode 150**: Blind 75 plus 75 more, grouped into pattern categories with video solutions.
- Time-box: 20–30 minutes stuck → read the approach (not the code), then implement it yourself and redo it a few days later (spaced repetition).
- Do mock interviews out loud. Solving silently and explaining live are different skills.

## Trade-offs & pitfalls
- Over-optimizing before you have a working solution. A correct O(n²) beats an unfinished O(n).
- Memorized solutions: interviewers change one constraint to test understanding.
- Ignoring the input constraints, which tell you the expected complexity.
- Writing code without agreeing on the approach, then having to throw it away.
- Not testing, or "testing" by re-reading instead of tracing.

## Senior perspective
- At senior level the algorithm round is a **floor**, not the differentiator: you're expected to solve it cleanly and quickly, then go deeper on follow-ups (scale, concurrency, memory, API design, how you'd test it in production).
- Seniors **drive the conversation**: they propose the constraints to assume, name the trade-offs unprompted, and keep time.
- Code quality signals seniority: small functions, meaningful names, input validation where it matters, no clever one-liners that hide bugs.
- Know when *not* to hand-roll: in real code you'd use a library heap, a database index, or a sorted container, and you should say so while still being able to implement it.

<Info>
**🆕 2025–2026: coding interviews and AI**

- **Meta** told employees in July 2025 it was building an "AI-enabled" coding interview where candidates have an AI assistant, calling it closer to the real developer environment and a way to make LLM-based cheating less effective ([404 Media](https://www.404media.co/meta-is-going-to-let-job-candidates-use-ai-during-coding-tests/)). The pilot started in **October 2025**: 60 minutes in CoderPad with a built-in assistant and a multi-file codebase, working through bug fixing, implementing a feature and optimizing it; it replaces one of the two onsite coding rounds ([Hello Interview](https://www.hellointerview.com/blog/meta-ai-enabled-coding)).
- **Canva** (engineering blog, 2025-06-11) replaced its Computer Science Fundamentals screen with an **AI-Assisted Coding** interview where candidates must use tools like Copilot, Cursor or Claude on larger, ambiguous problems, and are judged on breaking down requirements, reviewing and fixing AI-generated code, and production quality ([Canva](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/)).
- **Amazon** (2025) told recruiters candidates can be disqualified for using GenAI tools in interviews unless explicitly permitted ([GeekWire](https://www.geekwire.com/2025/is-it-cheating-ai-use-during-job-interviews-sparks-debate-over-whether-to-restrict-emerging-tools/)). **Anthropic** lets applicants use Claude to polish application materials but not in live interviews or take-homes unless told otherwise ([Fortune, 2025-07-21](https://fortune.com/2025/07/21/billion-dollar-giant-anthropic-ai-ban-hiring-policy-change-job-seekers-interview-process)).
- **Google**'s CEO said in 2025 that Google is reintroducing at least one in-person interview round, partly to make sure candidates have the fundamentals ([Business Standard, 2025-08-26](https://www.business-standard.com/companies/news/google-ai-cheating-job-interviews-in-person-hiring-shift-sundar-pichai-125082600492_1.html)).

**What this means for prep**: policies differ per company, so always ask what's allowed. The same UMPIRE skills apply in AI-enabled rounds, with more weight on **reading unfamiliar code, debugging, writing tests, and verifying** what the assistant produces instead of accepting it. Practice fixing bugs in someone else's code, not just writing from scratch.
</Info>

## Interview questions

<Accordion title="What should you do in the first five minutes of a coding interview?">
Restate the problem, ask clarifying questions about input size, types, edge cases and expected output, and write 2–3 examples including an edge case. Don't code yet.
</Accordion>

<Accordion title="Why state a brute-force solution if you know it's too slow?">
It shows you understand the problem, gives a correctness baseline and a fallback, and makes the bottleneck visible, which is usually where the optimization comes from.
</Accordion>

<Accordion title="How do you use input constraints to pick an approach?">
Map `n` to an affordable complexity: n ≤ 20 allows O(2ⁿ) backtracking, n ≤ 10⁴ allows O(n²), n ≤ 10⁶ needs O(n log n) or O(n). Then pick patterns with that complexity.
</Accordion>

<Accordion title="How do you test your solution without running it?">
Trace it on a small example, tracking variable values in a table, then walk through edge cases (empty, one element, duplicates, negatives, extremes). Look specifically for off-by-one and pointer-update bugs.
</Accordion>

<Accordion title="What do you do when you're stuck?">
Say what you're thinking, go back to a concrete example and solve it by hand, try a simpler version (sorted input, smaller n), list candidate data structures, and ask whether a hint is available. Don't go silent.
</Accordion>

<Accordion title="How would you approach an AI-assisted coding interview?">
Same framework: clarify and plan yourself, then use the assistant for well-scoped tasks (boilerplate, a helper, a test), read and verify every suggestion, run tests, and explain why you accept or reject its output. Interviewers score your judgment and verification, not the prompt.
</Accordion>

## Related
- [DSA Index](/Software%20Engineering/DSA/DSA%20Index) · [Big-O & Complexity Analysis](/Software%20Engineering/DSA/01%20Foundations/Big-O%20%26%20Complexity%20Analysis)
- Patterns: [Two Pointers](/Software%20Engineering/DSA/04%20Patterns/Two%20Pointers) · [Sliding Window](/Software%20Engineering/DSA/04%20Patterns/Sliding%20Window) · [Prefix Sum](/Software%20Engineering/DSA/04%20Patterns/Prefix%20Sum) · [Binary Search](/Software%20Engineering/DSA/04%20Patterns/Binary%20Search) · [Fast and Slow Pointer](/Software%20Engineering/DSA/04%20Patterns/Fast%20and%20Slow%20Pointer) · [Monotonic Stack & Queue](/Software%20Engineering/DSA/04%20Patterns/Monotonic%20Stack%20%26%20Queue) · [Top-K & Heaps pattern](/Software%20Engineering/DSA/04%20Patterns/Top-K%20%26%20Heaps%20pattern) · [Intervals](/Software%20Engineering/DSA/04%20Patterns/Intervals) · [Graph BFS-DFS patterns](/Software%20Engineering/DSA/04%20Patterns/Graph%20BFS-DFS%20patterns) · [Backtracking](/Software%20Engineering/DSA/04%20Patterns/Backtracking) · [Greedy](/Software%20Engineering/DSA/04%20Patterns/Greedy) · [Bit Manipulation](/Software%20Engineering/DSA/04%20Patterns/Bit%20Manipulation)
- [LeetCode Crash Course](/Software%20Engineering/DSA/05%20Books%20%26%20Courses/LeetCode%20Crash%20Course) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index) (for system design rounds)

## Sources
- [CodePath: UMPIRE Interview Strategy](https://guides.codepath.org/compsci/UMPIRE-Interview-Strategy)
- [Tech Interview Handbook: Coding interview techniques](https://www.techinterviewhandbook.org/coding-interview-techniques/) and [Grind 75](https://www.techinterviewhandbook.org/grind75)
- [NeetCode 150](https://neetcode.io/practice/practice/neetcode150) · [Blind 75 on NeetCode](https://neetcode.io/practice/practice/blind75)
- Gayle Laakmann McDowell, *Cracking the Coding Interview*, 6th ed., CareerCup 2015
- [404 Media: Meta Is Going to Let Job Candidates Use AI During Coding Tests (2025)](https://www.404media.co/meta-is-going-to-let-job-candidates-use-ai-during-coding-tests/)
- [Hello Interview: Meta's AI-Enabled Coding Interview](https://www.hellointerview.com/blog/meta-ai-enabled-coding)
- [Canva Engineering: Yes, You Can Use AI in Our Interviews (2025-06-11)](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/)
- [GeekWire: AI use during job interviews sparks debate (2025)](https://www.geekwire.com/2025/is-it-cheating-ai-use-during-job-interviews-sparks-debate-over-whether-to-restrict-emerging-tools/)
- [Fortune: Anthropic changes AI policy for job seekers (2025-07-21)](https://fortune.com/2025/07/21/billion-dollar-giant-anthropic-ai-ban-hiring-policy-change-job-seekers-interview-process)
- [Business Standard: Google to bring back in-person hiring (2025-08-26)](https://www.business-standard.com/companies/news/google-ai-cheating-job-interviews-in-person-hiring-shift-sundar-pichai-125082600492_1.html)
