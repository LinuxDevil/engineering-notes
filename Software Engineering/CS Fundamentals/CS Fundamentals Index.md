---
title: "CS Fundamentals Index"
tags: [cs-fundamentals, index]
aliases: [CS Fundamentals, Computer Science Fundamentals, CS Basics, CS Home]
updated: 2026-09-27
---
# CS Fundamentals Index

<Note>
**Start here**

The computer science underneath everything else in this vault: how the internet and network protocols work, what the operating system does for your program, how CPUs and memory really behave, how concurrency goes wrong, how text and data are encoded, the cryptography you use every day, and how code gets compiled or interpreted. Each note has worked examples, a 🆕 2025–2026 box where something changed, and interview questions. Algorithms and data structures live in the DSA folder.
</Note>

{/* diagram source: /diagrams/diagram-097.mmd */}
![diagram](/diagrams/diagram-097.svg)

## 🟢 Beginner
| Note | What you learn |
|---|---|
| [How the Internet Works](/Software%20Engineering/CS%20Fundamentals/How%20the%20Internet%20Works) | IP, TCP vs UDP, DNS resolution, what happens when you type a URL |
| [Encoding, Serialization & Character Sets](/Software%20Engineering/CS%20Fundamentals/Encoding%2C%20Serialization%20%26%20Character%20Sets) | ASCII, Unicode, UTF-8, Base64, JSON vs Protobuf vs Avro |
| [Compilers & Interpreters Basics](/Software%20Engineering/CS%20Fundamentals/Compilers%20%26%20Interpreters%20Basics) | Lexing, parsing, AST, JIT vs AOT |

## 🟡 Intermediate
| Note | What you learn |
|---|---|
| [Networking - OSI, TCP-IP & Protocols](/Software%20Engineering/CS%20Fundamentals/Networking%20-%20OSI%2C%20TCP-IP%20%26%20Protocols) | Layers, TCP handshake, congestion control, TLS 1.3, HTTP/1.1 vs 2 vs 3, QUIC |
| [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials) | Processes vs threads, scheduling, virtual memory, syscalls, epoll, io_uring |
| [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers) | Hashing vs encryption, password hashing, signatures, TLS certs, post-quantum |

## 🔴 Senior
| Note | What you learn |
|---|---|
| [Concurrency Primitives](/Software%20Engineering/CS%20Fundamentals/Concurrency%20Primitives) | Mutexes, semaphores, deadlocks, atomics, CAS, lock-free, memory ordering |
| [Computer Architecture Basics](/Software%20Engineering/CS%20Fundamentals/Computer%20Architecture%20Basics) | Memory hierarchy, cache lines, locality, false sharing, Amdahl's law |

Related elsewhere: [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism) (Principles) · [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) (Frontend) · [Event Loop - Microtasks & Macrotasks](/Software%20Engineering/Frontend/01%20JavaScript/Event%20Loop%20-%20Microtasks%20%26%20Macrotasks) · [Memory Management & Garbage Collection](/Software%20Engineering/Frontend/01%20JavaScript/Memory%20Management%20%26%20Garbage%20Collection) · [Node JS Under the hood!](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node%20JS%20Under%20the%20hood!)

<Info>
**🧭 Suggested study path**

1. [How the Internet Works](/Software%20Engineering/CS%20Fundamentals/How%20the%20Internet%20Works) → [Networking - OSI, TCP-IP & Protocols](/Software%20Engineering/CS%20Fundamentals/Networking%20-%20OSI%2C%20TCP-IP%20%26%20Protocols) → [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)
2. [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials) → [Concurrency And Parallelism](/Software%20Engineering/Principles/Concurrency%20And%20Parallelism) → [Concurrency Primitives](/Software%20Engineering/CS%20Fundamentals/Concurrency%20Primitives)
3. [Computer Architecture Basics](/Software%20Engineering/CS%20Fundamentals/Computer%20Architecture%20Basics) (why locality beats big-O constants)
4. [Encoding, Serialization & Character Sets](/Software%20Engineering/CS%20Fundamentals/Encoding%2C%20Serialization%20%26%20Character%20Sets) → [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)
5. [Compilers & Interpreters Basics](/Software%20Engineering/CS%20Fundamentals/Compilers%20%26%20Interpreters%20Basics) → the JS engine notes
6. Then apply it all in [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index)
</Info>

## 🆕 2025–2026 in this area
- **Post-quantum TLS by default**: ML-KEM (FIPS 203, 2024-08-13); Chrome 131 hybrid X25519MLKEM768 (Nov 2024); OpenSSL 3.5 default key share (2025-04-08); 52% of human traffic on Cloudflare PQ-encrypted by end of 2025; NIST picks HQC (2025-03-11) → [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers), [Networking - OSI, TCP-IP & Protocols](/Software%20Engineering/CS%20Fundamentals/Networking%20-%20OSI%2C%20TCP-IP%20%26%20Protocols)
- **Certificate lifetimes**: 200 days from 2026-03-15, 47 days by 2029 (CA/B Forum SC-081v3); Let's Encrypt ends OCSP (2025) and ships 6-day certs GA (Jan 2026) → [Cryptography Basics for Engineers](/Software%20Engineering/CS%20Fundamentals/Cryptography%20Basics%20for%20Engineers)
- **HTTP/3** ~21% of Cloudflare requests in 2025 → [Networking - OSI, TCP-IP & Protocols](/Software%20Engineering/CS%20Fundamentals/Networking%20-%20OSI%2C%20TCP-IP%20%26%20Protocols), [How the Internet Works](/Software%20Engineering/CS%20Fundamentals/How%20the%20Internet%20Works)
- **PostgreSQL 18** (2025-09-25) async I/O with io_uring; **JEP 491** (Java 24, March 2025) → [Operating Systems Essentials](/Software%20Engineering/CS%20Fundamentals/Operating%20Systems%20Essentials)
- **Python 3.14** (2025-10-07): free-threaded build officially supported, experimental JIT in official binaries → [Concurrency Primitives](/Software%20Engineering/CS%20Fundamentals/Concurrency%20Primitives), [Compilers & Interpreters Basics](/Software%20Engineering/CS%20Fundamentals/Compilers%20%26%20Interpreters%20Basics)
- **Node.js type stripping** on by default (v23.6.0 / v22.18.0), stable in v25.2.0 / v24.12.0 → [Compilers & Interpreters Basics](/Software%20Engineering/CS%20Fundamentals/Compilers%20%26%20Interpreters%20Basics)
- **Unicode 17.0** (2025-09-09) → [Encoding, Serialization & Character Sets](/Software%20Engineering/CS%20Fundamentals/Encoding%2C%20Serialization%20%26%20Character%20Sets)
- DNS as a single point of failure: AWS us-east-1 (2025-10-20) → [How the Internet Works](/Software%20Engineering/CS%20Fundamentals/How%20the%20Internet%20Works)

## Related
- [Software Engineering](/Software%20Engineering/Software%20Engineering), the vault root
- [System Design Index](/Software%20Engineering/System%20Design/System%20Design%20Index) · [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home)
