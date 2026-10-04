---
title: "Computer Architecture Basics"
tags: [cs-fundamentals, cs-fundamentals/architecture]
aliases: [Computer Architecture, CPU Caches, Memory Hierarchy, Data Locality, Cache Lines, False Sharing]
updated: 2026-09-27
---
# Computer Architecture Basics

<Note>
**TL;DR**

CPUs are much faster than memory, so hardware hides the gap with a **memory hierarchy**: registers → L1 → L2 → L3 caches → DRAM → SSD → network, each level ~10× bigger and slower. Data moves in **64-byte cache lines**, and **prefetchers** reward predictable, sequential access. So **data locality** (spatial: use neighbors; temporal: reuse soon) often matters more than big-O for real performance: arrays beat linked lists, column stores beat row stores for analytics, and time-ordered keys beat random ones in B-trees.
</Note>

## The CPU in one paragraph
A core fetches instructions, decodes them, and executes them in a **pipeline** (many instructions in flight). Modern cores are **superscalar** and **out-of-order** (execute independent instructions in parallel), use **branch prediction** to guess `if` outcomes (a mispredict costs ~15–20 cycles), and have **SIMD** units (AVX, NEON) that process several values per instruction. At ~3–5 GHz, one cycle is ~0.2–0.3 ns.

## The memory hierarchy
| Level | Typical size | Latency | Notes |
|---|---|---|---|
| Registers | ~KB | ~0 cycles | Compiler-managed |
| L1 cache (per core) | 32–64 KB data + instr | ~1 ns (4–5 cycles) | |
| L2 cache (per core) | 256 KB–2 MB | ~3–5 ns | |
| L3 cache (shared) | 8–100+ MB | ~10–20 ns | Shared across cores on a socket |
| DRAM | GBs–TBs | ~80–100 ns | Other socket's memory slower (NUMA) |
| NVMe SSD | TBs | ~20–100 µs random read | 4 KB pages |
| HDD | TBs | ~5–10 ms seek | Sequential is fine, random is terrible |
| Network (same DC) | — | ~0.5 ms RTT | |

A DRAM miss costs ~100× an L1 hit; that's hundreds of instructions the core could have run. Full latency table: [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework).

```mermaid
flowchart LR
  R["Registers"] --> L1["L1 ~1 ns"] --> L2["L2 ~4 ns"] --> L3["L3 ~15 ns"] --> RAM["DRAM ~100 ns"] --> SSD["SSD ~50 µs"] --> NET["Network ~0.5 ms+"]
```

## Cache lines and locality
- Memory moves between RAM and caches in **cache lines** (64 bytes on x86 and most ARM; 128 B on Apple M-series L2). Reading one byte fetches its whole line.
- **Spatial locality**: after touching address X, you'll likely touch X+8 → it's already in the line.
- **Temporal locality**: you'll touch X again soon → keep it cached.
- **Hardware prefetchers** detect sequential or strided patterns and fetch ahead. Random pointer chasing defeats them.

<Info>
**Row-major traversal matters**

```c
// 4096 x 4096 ints = 64 MB, bigger than any cache
for (i = 0; i < N; i++) for (j = 0; j < N; j++) sum += a[i][j];   // sequential: fast
for (j = 0; j < N; j++) for (i = 0; i < N; i++) sum += a[i][j];   // stride 16 KB: often 5-10x slower
```
Same O(N²), very different speed: the second loop uses one int per fetched cache line.
</Info>

<Info>
**Array vs linked list**

Summing 10 M integers in a contiguous array streams memory at full bandwidth with prefetching and SIMD. A linked list with nodes scattered on the heap incurs a potential cache miss per node, often **10–50× slower**, despite both being O(n).
</Info>

## Why this matters for backend engineers
| Where | Locality effect |
|---|---|
| **B-tree indexes** | Pages are sized for disk/cache; sequential keys (UUIDv7, auto-increment) keep hot pages cached ([Unique ID Generation](/Software%20Engineering/System%20Design/Unique%20ID%20Generation)) |
| **Column stores** (ClickHouse, Parquet) | Scanning one column reads contiguous, compressible values ([Row Vs Column Oriented Databases](/Software%20Engineering/Databases/Database%20Round/Row%20Vs%20Column%20Oriented%20Databases)) |
| **Hash maps** | Open addressing (contiguous) often beats chaining (pointers) |
| **Serialization** | Compact binary formats mean fewer bytes to move ([Encoding, Serialization & Character Sets](/Software%20Engineering/CS%20Fundamentals/Encoding%2C%20Serialization%20%26%20Character%20Sets)) |
| **GC languages** | Object-heavy structures scatter memory; value types/arrays of primitives help |
| **Batching** | Processing items in batches keeps code and data hot in cache |

## Multicore effects
- **Cache coherence** (MESI protocol) keeps cores' caches consistent: writing a line invalidates it in other cores.
- **False sharing**: two threads write different variables that sit on the **same cache line** → the line ping-pongs between cores, killing performance. Fix by padding/aligning per-thread data to 64 B (`@Contended` in Java, `alignas(64)` in C++). See [Concurrency Primitives](/Software%20Engineering/CS%20Fundamentals/Concurrency%20Primitives).
- **NUMA**: on multi-socket servers, memory attached to the other socket is slower; pin threads and memory together for latency-critical services.
- **Amdahl's law**: speedup ≤ 1 / (serial fraction + parallel fraction / cores). 10% serial caps speedup at 10× no matter how many cores.

## Storage hardware in brief
- **SSD**: no seek, fast random reads, but writes happen in pages and erases in large blocks → write amplification; sequential writes still preferred (LSM-trees exploit this).
- **HDD**: seek-dominated; sequential throughput decent. Still used for cheap bulk/cold storage.
- **fsync** latency (µs on enterprise NVMe with power-loss protection, ms on consumer drives) bounds commit throughput unless you group commits.

## Senior perspective
- Profile before optimizing: `perf stat` shows cache misses, branch misses, IPC (instructions per cycle). IPC < 1 on a hot loop usually means memory stalls.
- Data layout is a design decision: struct-of-arrays vs array-of-structs, fixed-size records, avoiding pointer-heavy structures in hot paths.
- In the cloud, vCPUs are hyperthreads; noisy neighbors share L3 and memory bandwidth.
- Big-O is necessary but not sufficient; constants from the memory hierarchy decide real throughput.

## Interview questions
<Accordion title="Why is iterating an array faster than a linked list with the same O(n)?">
Arrays are contiguous, so each 64-byte cache line brings several elements and the prefetcher streams ahead. Linked list nodes are scattered, so each node can be a cache miss costing ~100 ns.
</Accordion>

<Accordion title="What is a cache line and why does it matter?">
The unit of transfer between memory and CPU caches, typically 64 bytes. Accessing one byte loads the line, so accessing neighbors is nearly free and scattered access wastes bandwidth. It's also the unit of coherence, causing false sharing.
</Accordion>

<Accordion title="What is false sharing?">
Different threads writing independent variables that share a cache line; the coherence protocol bounces the line between cores so they effectively serialize. Fix by padding or separating per-thread data.
</Accordion>

<Accordion title="Why are random UUIDs bad B-tree keys?">
Inserts land on random leaf pages, so the working set is the whole index: more cache/buffer misses, page splits and I/O. Time-ordered keys append to the rightmost pages that stay hot in memory.
</Accordion>

<Accordion title="What does Amdahl's law say?">
The maximum speedup from parallelism is limited by the serial portion: speedup = 1 / (s + (1 − s)/N). With 5% serial work you can never exceed 20×.
</Accordion>

## Related
- [CS Fundamentals Index](/Software%20Engineering/CS%20Fundamentals/CS%20Fundamentals%20Index)
- [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials) · [Concurrency Primitives](/Software%20Engineering/CS%20Fundamentals/Concurrency%20Primitives) · [Compilers & Interpreters Basics](/Software%20Engineering/CS%20Fundamentals/Compilers%20%26%20Interpreters%20Basics)
- [System Design Interview Framework](/Software%20Engineering/System%20Design/System%20Design%20Interview%20Framework) · [Unique ID Generation](/Software%20Engineering/System%20Design/Unique%20ID%20Generation) · [Row Vs Column Oriented Databases](/Software%20Engineering/Databases/Database%20Round/Row%20Vs%20Column%20Oriented%20Databases) · [Database Pages](/Software%20Engineering/Databases/Database%20Round/Database%20Pages) · [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection)

## Sources
- Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*; Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*
- [Ulrich Drepper: What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Colin Scott: Latency numbers every programmer should know](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- [Martin Thompson: Mechanical Sympathy blog (false sharing)](https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html)
