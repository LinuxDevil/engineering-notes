---
tags: [cs-fundamentals, cs-fundamentals/encoding]
aliases: [Unicode, UTF-8, ASCII, Base64, Serialization, Protobuf, Avro, MessagePack, Character Encoding]
updated: 2026-09-27
---
# Encoding, Serialization & Character Sets

> [!tldr] TL;DR
> Computers store bytes; **character encodings** map text to bytes. **ASCII** covers 128 characters in 7 bits; **Unicode** assigns a **code point** to every character (~160k), and **UTF-8** encodes them in 1–4 bytes while staying ASCII-compatible: it's the web's default. **Base64** turns arbitrary bytes into safe ASCII text (+33% size); it is **not** encryption. **Serialization** turns in-memory objects into bytes: **JSON** (text, human-readable, schemaless), **Protobuf** (binary, schema with field numbers, great for gRPC), **Avro** (binary, schema travels with data or registry, great for Kafka and data lakes), **MessagePack** (binary JSON). Pick based on schema evolution, size/speed, and who reads it.

## Characters
### ASCII and code pages
ASCII: 0–127 (`A` = 65 = `0x41`). The 128–255 range meant different things in different "code pages" (Latin-1, Windows-1252), which caused mojibake (`CafÃ©` instead of `Café`).

### Unicode
- A **code point** is a number: `U+0041` A, `U+00E9` é, `U+4E2D` 中, `U+1F600` 😀.
- Code points ≠ what users see. A **grapheme cluster** (user-perceived character) can be several code points: `é` can be `U+00E9` or `e` + combining accent `U+0301`; 👩‍💻 is woman + ZWJ + laptop; flags are two regional indicators.
- **Normalization** (NFC/NFD/NFKC) makes equivalent strings byte-identical. Normalize before comparing or using as keys (usernames!).

### UTF-8, UTF-16, UTF-32
| Code point range | UTF-8 bytes | Example |
|---|---|---|
| U+0000–U+007F | 1 | `A` → `41` |
| U+0080–U+07FF | 2 | `é` → `C3 A9` |
| U+0800–U+FFFF | 3 | `中` → `E4 B8 AD` |
| U+10000–U+10FFFF | 4 | `😀` → `F0 9F 98 80` |

- **UTF-8**: variable length, ASCII-compatible, no byte-order issues, self-synchronizing. Default for the web, JSON (RFC 8259 requires it for interchange), Linux, Go, Rust.
- **UTF-16**: 2 or 4 bytes (surrogate pairs for > U+FFFF). Used internally by JavaScript, Java, C#, Windows APIs. That's why `"😀".length === 2` in JS.
- **UTF-32**: fixed 4 bytes; simple indexing, wasteful.

```js
const s = "héllo 😀";
s.length;                           // 8  (UTF-16 code units)
[...s].length;                      // 7  (code points)
new TextEncoder().encode(s).length; // 11 (UTF-8 bytes)
[...new Intl.Segmenter().segment("👩‍💻")].length; // 1 grapheme
```

> [!warning] Practical pitfalls
> - "Length" is ambiguous: bytes (storage limits), code units (JS/Java), code points, graphemes (UI). Truncating by bytes or code units can cut a character in half.
> - MySQL `utf8` is a 3-byte subset; use **`utf8mb4`** for emoji.
> - Always declare encodings: `Content-Type: application/json; charset=utf-8`, `<meta charset="utf-8">`.
> - Homoglyph attacks (`раураl.com` with Cyrillic letters): normalize and restrict scripts for identifiers.

## Binary-to-text encodings
| Encoding | Alphabet | Overhead | Use |
|---|---|---|---|
| **Hex** | 0-9a-f | 100% | Hashes, debugging |
| **Base64** | A–Z a–z 0–9 + / (=) | ~33% | Email attachments, data URLs, embedding bytes in JSON, JWT (Base64url) |
| **Base64url** | `-` `_` instead of `+` `/`, no padding | ~33% | URLs, JWTs |
| **Base32** | A–Z 2–7 | 60% | Case-insensitive contexts, TOTP secrets, ULIDs (Crockford) |
| **Base62** | 0-9 a-z A-Z | — | Short IDs ([[Case Study - URL Shortener]]) |
| **Percent-encoding** | `%20` | varies | URLs |

```bash
echo -n 'hi!' | base64          # aGkh
echo 'aGkh' | base64 -d         # hi!
```
Base64 takes each 3 bytes (24 bits) → 4 characters of 6 bits. **Encoding ≠ encryption**: anyone can decode it ([[Cryptography Basics for Engineers]]).

## Serialization formats
| Format | Type | Schema | Size/speed | Evolution | Typical use |
|---|---|---|---|---|---|
| **JSON** | Text | Optional (JSON Schema) | Larger, slower parse | Add fields freely; no types for dates/bytes/int64 | Public REST APIs, config |
| **Protobuf** | Binary | `.proto`, required | Small, fast | Field **numbers** identify fields; add/remove optional fields safely | gRPC, internal services ([[gRPC vs REST]]) |
| **Avro** | Binary | JSON-defined schema; writer + reader schemas resolved | Very compact (no field tags) | Rich rules; schema registry | Kafka, Hadoop/data lakes ([[Message Queues & Event Streaming]]) |
| **MessagePack / CBOR** | Binary | None | Smaller than JSON | Like JSON | Caches, IoT, Redis values |
| **Parquet / ORC** | Columnar binary files | Schema | Very compact for analytics | Column evolution | Data warehouses/lakes |
| XML | Text | XSD | Verbose | Namespaces | Legacy, SOAP, documents |

```protobuf
syntax = "proto3";
message User {
  int64 id = 1;
  string email = 2;
  repeated string roles = 3;
  optional string display_name = 4;   // added later: old readers ignore field 4
  reserved 5;                         // removed field: never reuse number 5
}
```
Protobuf wire format: each field = (field number, wire type) tag + value, varint-encoded integers. `id=150` is 3 bytes (`08 96 01`) vs `"id":150` = 8 bytes in JSON.

### Schema evolution rules of thumb
- **Backward compatible** (new readers read old data) and **forward compatible** (old readers read new data): add optional fields with defaults; never change a field's type or number; never reuse removed numbers/names.
- Enforce with a schema registry (Confluent, Apicurio) and CI checks (`buf breaking`).
- JSON: tolerant readers (ignore unknown fields), explicit versioning in URLs or media types for breaking changes ([[REST API Design]]).

### Gotchas
- JSON numbers above 2⁵³ lose precision in JavaScript: send 64-bit IDs as strings.
- Dates: use ISO 8601 with timezone (`2026-09-27T10:00:00Z`) or epoch millis; never locale strings.
- **Deserialization of untrusted data** into arbitrary types (Java native serialization, Python `pickle`, YAML with tags) enables remote code execution. Use data-only formats and allowlists.

## Senior perspective
- Choose formats per boundary: JSON at public edges (debuggability, ecosystem), Protobuf between services, Avro/Protobuf with a registry on event streams, Parquet for analytics.
- Payload size matters at scale: compression (gzip, zstd, Brotli) often closes most of the JSON vs binary gap for storage; CPU for parsing remains.
- Text handling is an i18n and security issue: normalize identifiers, validate UTF-8 at boundaries, store `utf8mb4`/`text` everywhere.

> [!info]+ 🆕 2025–2026
> **Unicode 17.0 (2025-09-09)** added 4,803 characters (total 159,801), four new scripts, CJK Extension J and the Saudi Riyal sign; new emoji need font and OS updates before users see them, so never assume a client can render the latest code points. ([Unicode blog](https://blog.unicode.org/2025/09/unicode-170-release-announcement.html))

## Interview questions
> [!question]- Unicode vs UTF-8?
> Unicode is the character set: it assigns code points to characters. UTF-8 is one encoding of those code points into bytes (1–4 bytes each, ASCII-compatible). UTF-16 and UTF-32 are others.

> [!question]- Why does "😀".length return 2 in JavaScript?
> JS strings are UTF-16 code units; code points above U+FFFF are stored as a surrogate pair of two units. Use `[...str]` for code points or `Intl.Segmenter` for graphemes.

> [!question]- Is Base64 encryption?
> No. It's a reversible encoding of bytes into ASCII text with no key. It provides zero confidentiality; use encryption for secrecy.

> [!question]- Protobuf vs JSON?
> Protobuf is binary, schema-first, smaller and faster, with safe evolution via field numbers, but not human-readable and needs generated code. JSON is text, self-describing, universal and debuggable but larger, slower and weakly typed.

> [!question]- Why is Avro popular with Kafka?
> Very compact records (no per-field tags), and schema evolution via writer/reader schema resolution with a schema registry that stores schemas by ID, so each message carries just a small schema ID.

> [!question]- What makes a schema change backward compatible?
> New code can read old data: adding optional fields with defaults, not removing required fields, not changing types or field numbers, not reusing removed identifiers.

## Related
- [[CS Fundamentals Index]]
- [[Cryptography Basics for Engineers]] · [[Computer Architecture Basics]]
- [[gRPC vs REST]] · [[REST API Design]] · [[Message Queues & Event Streaming]] · [[Unique ID Generation]] · [[JWT vs Cookies]]

## Sources
- [The Unicode Standard](https://www.unicode.org/standard/standard.html) · [Unicode 17.0 announcement](https://blog.unicode.org/2025/09/unicode-170-release-announcement.html)
- [RFC 3629: UTF-8](https://www.rfc-editor.org/rfc/rfc3629) · [RFC 4648: Base16/32/64](https://www.rfc-editor.org/rfc/rfc4648) · [RFC 8259: JSON](https://www.rfc-editor.org/rfc/rfc8259)
- [Joel Spolsky: The Absolute Minimum Every Software Developer Must Know About Unicode](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
- [Protocol Buffers: encoding](https://protobuf.dev/programming-guides/encoding/) · [Apache Avro spec](https://avro.apache.org/docs/current/specification/) · [MessagePack spec](https://github.com/msgpack/msgpack/blob/master/spec.md)
- Martin Kleppmann, *Designing Data-Intensive Applications*, ch. 4 "Encoding and Evolution"
- [OWASP: Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)
