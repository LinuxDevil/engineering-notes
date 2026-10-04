---
title: "Java Language Essentials"
tags: [backend, backend/java]
aliases: [Java Basics, Java Fundamentals, Java Collections, Java Generics, Java Streams, Records, Sealed Classes, Java Exceptions]
updated: 2026-09-27
---
# Java Language Essentials

<Note>
**TL;DR**

Java is a statically typed, object-oriented language compiled to **bytecode** that runs on the **JVM**. The daily toolkit is: primitives vs reference types, classes/interfaces, the **Collections** framework (`List`, `Set`, `Map`), **generics**, **exceptions**, **lambdas + Streams**, and since Java 16–21 **records**, **sealed types** and **pattern matching** that make data-oriented code concise. Know `equals`/`hashCode`, immutability and null handling cold; they're behind most real bugs.
</Note>

## How Java runs

{/* diagram source: /diagrams/diagram-042.mmd */}
![diagram](/diagrams/diagram-042.svg)

```java
// Java 25: compact source file, run with `java Hello.java`
void main() {
    IO.println("Hello, Java");
}
```

Classic form (still what you write in real projects):

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java");
    }
}
```

## Types

| Primitive | Size | Wrapper |
|---|---|---|
| `boolean` | JVM-dependent | `Boolean` |
| `byte`, `short`, `int`, `long` | 8, 16, 32, 64 bits | `Byte`, `Short`, `Integer`, `Long` |
| `float`, `double` | 32, 64 bits | `Float`, `Double` |
| `char` | 16 bits (UTF-16 code unit) | `Character` |

- Primitives hold values; everything else is a **reference** to an object on the heap. Java is always **pass-by-value** (for objects, the reference is copied).
- **Autoboxing** converts `int` ↔ `Integer` automatically. Pitfalls: `NullPointerException` when unboxing `null`, and `==` on boxed values compares references (`Integer` cache only covers -128..127).
- Use `BigDecimal` for money, never `double`.
- `var` (Java 10) infers local variable types: `var users = new ArrayList<User>();`.
- `String` is immutable; build strings in loops with `StringBuilder`. Text blocks (Java 15): `"""..."""`.

## Classes, interfaces, OOP

```java
public interface Shape { double area(); }

public abstract class Base implements Shape {
    protected final String name;
    protected Base(String name) { this.name = name; }
}

public final class Circle extends Base {
    private final double r;
    public Circle(double r) { super("circle"); this.r = r; }
    @Override public double area() { return Math.PI * r * r; }
}
```

- Access modifiers: `private` < package-private (default) < `protected` < `public`.
- `final` class can't be extended; `final` field can't be reassigned; `static` belongs to the class.
- Interfaces can have `default` and `static` methods (Java 8) and `private` methods (Java 9).
- Abstract class vs interface: [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) and [Reactive Spring - Ch 2 Java Refresher](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%202%20Java%20Refresher). OOP principles: [OOP](/Software%20Engineering/Principles/OOP), [SOLID](/Software%20Engineering/Principles/SOLID).

### `equals`, `hashCode`, `toString`

If two objects are `equals`, they **must** have the same `hashCode`, or `HashMap`/`HashSet` break. Records generate all three for you.

## Records, sealed types and pattern matching

```java
public record Money(BigDecimal amount, String currency) {
    public Money {                                   // compact constructor: validation
        if (amount.signum() < 0) throw new IllegalArgumentException("negative");
        Objects.requireNonNull(currency);
    }
}

public sealed interface Payment permits Card, BankTransfer, Wallet {}
public record Card(String last4) implements Payment {}
public record BankTransfer(String iban) implements Payment {}
public record Wallet(String provider) implements Payment {}

static String describe(Payment p) {
    return switch (p) {                              // exhaustive: no default needed
        case Card(var last4)          -> "card ****" + last4;
        case BankTransfer(var iban)   -> "bank " + iban;
        case Wallet w when w.provider().equals("apple") -> "Apple Pay";
        case Wallet w                 -> "wallet " + w.provider();
    };
}
```

- **Records** (Java 16): immutable data carriers with auto constructor, accessors, `equals/hashCode/toString`. Great for DTOs and value objects.
- **Sealed** types (Java 17): restrict which classes can implement/extend, so the compiler knows all cases.
- **Pattern matching**: `instanceof` patterns (16), `switch` patterns and record deconstruction (21), unnamed `_` patterns (22). Together these give algebraic data types in Java.

## Collections

{/* diagram source: /diagrams/diagram-043.mmd */}
![diagram](/diagrams/diagram-043.svg)

| Need | Use | Big-O (typical) |
|---|---|---|
| Indexed list | `ArrayList` | get O(1), add at end amortised O(1) |
| Unique items | `HashSet` | add/contains O(1) |
| Key lookup | `HashMap` | get/put O(1) |
| Sorted keys / range queries | `TreeMap` | O(log n) |
| Insertion order | `LinkedHashMap` / `LinkedHashSet` | O(1) |
| Stack / queue | `ArrayDeque` | O(1) at ends |
| Priority | `PriorityQueue` | O(log n) offer/poll |
| Thread-safe map | `ConcurrentHashMap` | O(1), lock striping |

```java
List<String> names = List.of("ana", "bo");          // immutable (throws on add)
var mutable = new ArrayList<>(names);
Map<String, Integer> counts = new HashMap<>();
counts.merge("ana", 1, Integer::sum);                 // increment
counts.computeIfAbsent("groups", k -> 0);
// Java 21 SequencedCollection
mutable.getFirst(); mutable.getLast(); mutable.reversed();
```

## Generics

```java
public class Box<T> { private final T value; public Box(T v) { value = v; } public T get() { return value; } }

static <T extends Comparable<T>> T max(List<T> xs) { return Collections.max(xs); }

// PECS: Producer Extends, Consumer Super
static double sum(List<? extends Number> nums) { return nums.stream().mapToDouble(Number::doubleValue).sum(); }
static void fill(List<? super Integer> out) { out.add(1); }
```

- Generics are checked at compile time and **erased** at runtime (`List<String>` and `List<Integer>` are the same class). You can't do `new T()` or `instanceof List<String>`.
- Avoid raw types (`List` without `<>`).

## Exceptions

{/* diagram source: /diagrams/diagram-044.mmd */}
![diagram](/diagrams/diagram-044.svg)

```java
try (var in = Files.newBufferedReader(path)) {        // try-with-resources closes automatically
    return in.readLine();
} catch (NoSuchFileException e) {
    throw new ConfigMissingException(path, e);       // wrap, keep the cause
} finally {
    metrics.increment("config.read");
}
```

- Checked exceptions must be caught or declared; unchecked don't. Modern frameworks (Spring) mostly use unchecked exceptions.
- Never swallow exceptions (`catch (Exception e) {}`); log or rethrow with context.
- Don't use exceptions for normal control flow.

## Lambdas, functional interfaces and Streams

```java
record Order(String customer, BigDecimal total, Status status) {}

Map<String, BigDecimal> revenueByCustomer = orders.stream()
    .filter(o -> o.status() == Status.PAID)
    .collect(Collectors.groupingBy(Order::customer,
             Collectors.reducing(BigDecimal.ZERO, Order::total, BigDecimal::add)));

List<String> top3 = revenueByCustomer.entrySet().stream()
    .sorted(Map.Entry.<String, BigDecimal>comparingByValue().reversed())
    .limit(3)
    .map(Map.Entry::getKey)
    .toList();                                        // Java 16
```

- Core functional interfaces: `Function<T,R>`, `Supplier<T>`, `Consumer<T>`, `Predicate<T>`, `BiFunction`, `UnaryOperator`.
- Streams are **lazy**: intermediate ops (`map`, `filter`) do nothing until a terminal op (`collect`, `toList`, `forEach`, `reduce`). Same idea as "nothing happens until you subscribe" in [Reactor - Mono & Flux](/Software%20Engineering/Backend/Java/Java%20Reactive%20Course/Reactor%20-%20Mono%20%26%20Flux).
- Streams are single-use. `parallelStream()` uses the common ForkJoinPool; rarely a win on servers.
- **Gatherers** (Java 24) add custom intermediate operations (`windowFixed`, `fold`, `mapConcurrent`).

### `Optional`

```java
Optional<User> user = repo.findByEmail(email);
String name = user.map(User::name).orElse("anonymous");
User u = user.orElseThrow(() -> new NotFoundException(email));
```

Use `Optional` as a **return type** for "maybe absent", not for fields or parameters.

## Pitfalls
- `==` vs `equals` on strings and boxed numbers.
- Mutating a collection while iterating it (`ConcurrentModificationException`); use `removeIf`.
- `HashMap` key objects whose `hashCode` changes after insertion (mutable keys).
- `List.of(...)` is immutable and rejects `null`.
- Integer overflow is silent; use `Math.addExact` or `long`.
- Returning `null` collections instead of empty ones.

## Senior perspective
- Prefer **immutability** (records, `List.copyOf`, final fields): thread safety and easier reasoning for free.
- Model domain states with **sealed types + records** instead of flags and nullable fields; exhaustive `switch` turns missing cases into compile errors.
- Adopt **JSpecify** `@Nullable`/`@NullMarked` with NullAway or IntelliJ to make nullness explicit (Spring 7 did portfolio-wide). See [Spring Boot 3 to 4 - What's New](/Software%20Engineering/Backend/Java/Spring%20Boot%203%20to%204%20-%20What's%20New).
- In review: `equals/hashCode` consistency, resource closing, exception wrapping with cause, streams that hide side effects, accidental O(n²) (`list.contains` in loops).

<Info>
**🆕 2025–2026**

- **JDK 25 (LTS, 2025-09-16)** finalised *compact source files and instance main methods* (JEP 512, `void main()` + `IO.println`), *module import declarations* (JEP 511, `import module java.base;`) and *flexible constructor bodies* (JEP 513: statements before `super(...)`).
- **JDK 24 (2025-03-18)** finalised **Stream Gatherers** (JEP 485).
- Primitive types in patterns (`case int i`) is still in preview (5th preview in JDK 27, 2026-09-15).
Full list: [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025).
</Info>

## Interview questions

<Accordion title="What's the contract between `equals` and `hashCode`?">
Equal objects must have equal hash codes (the reverse isn't required). Breaking it makes hash-based collections lose or duplicate entries. Override both together, based on the same fields; records do it automatically.
</Accordion>

<Accordion title="Checked vs unchecked exceptions?">
Checked (`Exception` subclasses except `RuntimeException`) must be caught or declared, for recoverable conditions like I/O. Unchecked (`RuntimeException`) signal programming errors or are used by frameworks to avoid boilerplate. `Error`s are JVM problems you shouldn't catch.
</Accordion>

<Accordion title="What is type erasure?">
Generic type parameters exist only at compile time; the compiler inserts casts and erases them to their bounds in bytecode. Consequences: no `new T()`, no generic arrays, no `instanceof List<String>`, and overloads can't differ only by type argument.
</Accordion>

<Accordion title="`ArrayList` vs `LinkedList`?">
`ArrayList` is a resizable array: O(1) random access, cache-friendly; inserts in the middle are O(n). `LinkedList` has O(1) insert/remove at a known node but O(n) access and poor locality. In practice `ArrayList` (or `ArrayDeque` for queues) almost always wins.
</Accordion>

<Accordion title="What are records and sealed classes good for?">
Records: concise immutable data (DTOs, value objects, events). Sealed classes: closed hierarchies so the compiler knows all subtypes. Together with pattern-matching `switch` they give exhaustive, type-safe handling of variants.
</Accordion>

<Accordion title="How does `HashMap` work internally?">
An array of buckets indexed by `hash(key) & (n-1)`. Collisions form a linked list that turns into a red-black tree when a bucket exceeds 8 entries (and the table is large enough). It resizes (doubles) when size exceeds capacity × load factor (0.75), rehashing entries.
</Accordion>

<Accordion title="Are Java streams lazy?">
Yes. Intermediate operations build a pipeline; nothing runs until a terminal operation. Elements flow one at a time through the whole pipeline, and short-circuiting ops like `findFirst`/`limit` stop early.
</Accordion>

## Related
- [Java & Spring Index](/Software%20Engineering/Backend/Java/Java%20%26%20Spring%20Index)
- [Modern Java 17 to 25](/Software%20Engineering/Backend/Java/Modern%20Java%2017%20to%2025) · [Java Concurrency & Virtual Threads](/Software%20Engineering/Backend/Java/Java%20Concurrency%20%26%20Virtual%20Threads) · [JVM Internals - Memory & Garbage Collection](/Software%20Engineering/Backend/Java/JVM%20Internals%20-%20Memory%20%26%20Garbage%20Collection)
- [Reactive Spring - Ch 2 Java Refresher](/Software%20Engineering/Backend/Java/Reactive%20Spring%20Book/Reactive%20Spring%20-%20Ch%202%20Java%20Refresher) · [OOP](/Software%20Engineering/Principles/OOP) · [SOLID](/Software%20Engineering/Principles/SOLID) · [Abstract Class and Interface](/Software%20Engineering/Principles/Abstract%20Class%20and%20Interface) · [Design Patterns](/Software%20Engineering/Principles/Design%20Patterns)

## Sources
- [dev.java: Learn Java](https://dev.java/learn/)
- [Java SE 25 API docs](https://docs.oracle.com/en/java/javase/25/docs/api/)
- [JEP 395: Records](https://openjdk.org/jeps/395) · [JEP 409: Sealed Classes](https://openjdk.org/jeps/409) · [JEP 441: Pattern Matching for switch](https://openjdk.org/jeps/441) · [JEP 440: Record Patterns](https://openjdk.org/jeps/440)
- [JEP 485: Stream Gatherers](https://openjdk.org/jeps/485) · [JEP 512: Compact Source Files and Instance Main Methods](https://openjdk.org/jeps/512)
- [OpenJDK: JDK 25](https://openjdk.org/projects/jdk/25/) · [OpenJDK: JDK 27](https://openjdk.org/projects/jdk/27/)
- Joshua Bloch, *Effective Java*, 3rd ed. (Addison-Wesley, 2018)
