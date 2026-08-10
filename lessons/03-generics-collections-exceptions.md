# Lesson 03 — Generics, Collections, Exceptions

**Date:** 2026-08-10 · **Phase:** 1 (Java language) · **Prereq:** Lesson 02 (streams, Optional)

## 1. The mental model that changes everything

You already use generics without naming them: `Optional<User>`, `List<String>`, `Map<String, Long>` from Lesson 02. This lesson makes them *explicit*, then covers the two things collections force you to decide (which implementation, how you read them), and finally the Java failure model — **exceptions**, which is where Java and TS diverge most.

Three mental shifts:

1. **Generics are about compile-time safety, and they're *invariant*.** In TS, `Array<Dog>` is assignable to `Array<Animal>` (structural typing). In Java, `List<Dog>` is **not** a `List<Animal>` — the compiler refuses, because `List<Animal>` would let you `add(new Cat())`. Invariance *looks* like a limitation; it's actually the compiler protecting you from type-unsafe writes.
2. **Collections: interface vs implementation.** `List` says *what* you can do; `ArrayList` / `LinkedList` say *how* it's stored. The interface is the contract you code against; the implementation is the performance decision.
3. **Exceptions are part of the type system.** In TS, `throw new Error()` is untyped — any function can throw anything, and the compiler doesn't care. In Java, some exceptions are **checked**: the compiler *forces* you to handle them. This is Java's way of making failure part of the API contract.

## 2. The syntax delta, side by side

| TS | Java |
|---|---|
| `Array<T>` | `List<T>` |
| `Set<T>` | `Set<T>` — impls: `HashSet`, `LinkedHashSet`, `TreeSet` |
| `Map<K, V>` | `Map<K, V>` — impls: `HashMap`, `LinkedHashMap`, `TreeMap` |
| `arr.push(x)` | `list.add(x)` |
| `arr.length` | `list.size()` |
| `arr.includes(x)` / `arr.indexOf(x)` | `list.contains(x)` / `list.indexOf(x)` |
| `arr[0]` | `list.get(0)` |
| `arr.at(-1)` | `list.get(list.size() - 1)` |
| `new Array(...)` / `[...]` | `List.of(...)` (immutable) / `new ArrayList<>(...)` |
| `obj[key]` / `Object.keys()` | `map.get(key)` / `map.keySet()` |
| `throw new Error("msg")` | `throw new IllegalArgumentException("msg")` |
| `try / catch / finally` | `try / catch / finally` + **try-with-resources** |
| `instanceof` (runtime check) | `instanceof` (works the same) |
| `Array.isArray(x)` | `x instanceof List` |

## 3. Generics — compile-time safety with one rule that breaks TS intuition

### Why they exist

Without generics, you'd need one class per type (or cast everywhere and pray). Generics let one class serve every type **with compile-time checking**:

```java
// One method, any type — type-checked at the call site
public <T> Optional<T> firstMatch(List<T> items, Predicate<T> predicate) {
    return items.stream().filter(predicate).findFirst();
}
```

- `<T>` before the return type *declares* the type parameter (like TS's `<T,>` on a function).
- Inside, `T` is a placeholder; at the call site the compiler checks: `firstMatch(List<User>, User -> boolean)` → `Optional<User>`.

### Internal mechanism: type erasure

Here's where Java differs from TS under the hood:

```java
List<String> names = new ArrayList<>();
List<Integer> nums = new ArrayList<>();
System.out.println(names.getClass() == nums.getClass()); // true!
```

- At runtime there's **one** class: `ArrayList`. The `<String>` / `<Integer>` is compile-time information only — erased.
- What the compiler actually emits: casts inserted at the boundaries. `String name = names.get(0)` becomes `String name = (String) names.get(0)`.
- **Consequence:** you can't do `if (x instanceof T)` or `new T()` — the type is gone at runtime.

TS erases *all* types (`interface` vanishes); Java erases generic type *arguments* but keeps the class hierarchy. Both end up as runtime casts, but Java inserts them automatically and checks them.

### The rule that matters: invariance (and its escape hatch)

```java
List<String> strings = new ArrayList<>();
List<Object> objects = strings; // ❌ compile error — why?
```

Because if it compiled, someone could `objects.add(42)` — and now a `List<String>` contains an `Integer`. TS allows this structurally (until a runtime surprise); Java refuses it at compile time.

The escape hatch when you need covariance (read-only): **bounded wildcards** — `? extends` / `? super` (PECS: **P**roducer **E**xtends, **C**onsumer **S**uper):

```java
// READING from a list of unknowns — safe: every element is at least a Number
double sum(List<? extends Number> numbers) {
    return numbers.stream().mapToDouble(Number::doubleValue).sum();
}
sum(new ArrayList<Integer>()); // ✅ Integer IS a Number, read-only OK

// WRITING — ? super allows adding Dog to a Dog-or-ancestor collection
void addDog(List<? super Dog> dogs) { dogs.add(new Dog()); }
addDog(new ArrayList<Animal>()); // ✅ Animal can hold a Dog
```

**Mental model:** `? extends T` = "I'll only read, so any subtype is fine"; `? super T` = "I'll only write, so any supertype that can hold a T is fine". You see `? extends` constantly in real signatures (e.g. `Collection<? extends T>`).

**Production rules:**
- Prefer the **diamond**: `new ArrayList<>()` — let the compiler infer the type.
- Declare variables with the interface type: `List<String>`, not `ArrayList<String>` (your callers shouldn't care about the implementation).
- Don't hand-write `? extends` in your own APIs unless you're building library-level code — in services, concrete generic types (`List<UrlStats>`) are what you actually want. Knowing the rules is for *reading* other people's signatures.

## 4. Collections — interface vs implementation

**Mental model:** the interface is a contract ("ordered, indexed sequence"), the implementation is a data structure with different perf. Choosing the implementation is a *performance decision about access patterns*.

### List — ordered, indexed, duplicates allowed

| Impl | Get by index | Insert/remove middle | Insert/remove ends | Memory |
|---|---|---|---|---|
| `ArrayList` (default) | O(1) | O(n) shift | O(1) at end, O(n) at front | compact array |
| `LinkedList` | O(n) walk | O(n) but cheap node ops | O(1) both ends | per-node overhead |

**When:** `ArrayList` is the default — contiguous reads and index access. `LinkedList` is for queue/deque behavior (though `ArrayDeque` beats it there). If you don't know why you'd pick `LinkedList`, use `ArrayList`.

### Set — unique, no index

| Impl | Order | Lookup | Use when |
|---|---|---|---|
| `HashSet` (default) | unordered | O(1) | "is this present?" — fastest |
| `LinkedHashSet` | insertion order | O(1) | need order + uniqueness |
| `TreeSet` | sorted (Comparable/Comparator) | O(log n) | need sorted unique values |

**Key trap:** `HashSet` iteration order is *not* guaranteed — never rely on it. If you need "the order I inserted" or "sorted", pick `LinkedHashSet`/`TreeSet` explicitly.

### Map — key → value

| Impl | Order | Lookup | Use when |
|---|---|---|---|
| `HashMap` (default) | unordered | O(1) | default key-value |
| `LinkedHashMap` | insertion order | O(1) | "keep my insertion order" |
| `TreeMap` | sorted by key | O(log n) | sorted keys / range queries |

**Production rules:**
- **Never return `null` for a collection** — return `List.of()` / `Set.of()` / `Map.of()` (empty, immutable). Null collections are where every downstream `.stream()` NPEs.
- `List.of(...)` is **immutable** — `add` throws `UnsupportedOperationException`. If you need to build then return, `new ArrayList<>(...)` then return it.
- **Thread-safety:** `ArrayList`/`HashMap` are not thread-safe. For reads after startup (immutable collections) it's fine; for concurrent writes use `ConcurrentHashMap` / `CopyOnWriteArrayList` (Lesson 5 will cover this properly).

## 5. Exceptions — failure as part of the contract

### The hierarchy

```
Throwable
├── Error            → JVM-level (OutOfMemoryError...). DO NOT catch.
└── Exception
    ├── RuntimeException → unchecked: compiler doesn't force handling
    │     (NullPointerException, IllegalArgumentException, IllegalStateException...)
    └── (checked exceptions) → compiler FORCES handling
          (IOException, SQLException, InterruptedException...)
```

### Checked vs unchecked — the difference from TS

```java
// CHECKED — the method signature *declares* it; callers must handle it
public String readFile(Path path) throws IOException {
    return Files.readString(path);      // IOException is checked
}
// Caller must: try/catch OR declare `throws IOException` too

// UNCHECKED — nothing forced, but it can still happen
public String getOrThrow(long id) {
    return repository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("no user " + id));
}
```

**Why Java has both:** checked exceptions force the *caller* to make an explicit decision about failure — either handle it here or declare "I pass the responsibility up". TS just crashes at runtime.

**The Spring/industry stance (important trade-off):** modern Java frameworks treat *most* application failures as **unchecked** (`RuntimeException`). Checked exceptions are for *external, recoverable* failures (I/O, network, malformed data) — and even those often get wrapped into unchecked exceptions at the service boundary. You will rarely declare `throws` in Spring code; you'll throw `RuntimeException` subclasses and let a global handler translate them.

### try-with-resources — the `finally` killer

TS has no equivalent; you manually close in `finally`. Java closes automatically:

```java
// Before (Java 6 style — you'll see it in old code):
BufferedReader r = null;
try {
    r = Files.newBufferedReader(path);
    // ...
} finally {
    if (r != null) r.close();   // manual, easy to forget
}

// Modern Java — resource auto-closed, even on exception:
try (var r = Files.newBufferedReader(path)) {
    return r.readLine();
}
// no finally needed — the try() block closes r for you
```

**Rule:** any object that holds a resource (stream, connection, file) and implements `AutoCloseable` goes in the `try (...)` parentheses.

### Custom exceptions — the production pattern

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(long id) {
        super("User not found: " + id);
    }
}
```

- Extend `RuntimeException` → unchecked → no forced `throws` everywhere.
- One class per domain failure; the class *name* carries meaning.
- Spring maps it to HTTP automatically (Lesson 3 of the web layer, but the shape is worth seeing now):

```java
@RestControllerAdvice
public class ApiExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleNotFound(UserNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

**Never swallow exceptions.** `catch (Exception e) {}` with an empty body is a bug — you've hidden the failure. If you catch, do something: log, translate, or rethrow (possibly wrapped).

## 6. Reading your own code — the homework from Lesson 02, upgraded

Your Lesson 02 homework was a CSV export. Here's the production-flavored version using all three tools:

```java
@Service
public class CsvExportService {

    private final UserRepository repository;          // ① injected

    public CsvExportService(UserRepository repository) {
        this.repository = repository;
    }

    public String exportActiveUsers() {               // ② never returns null
        List<User> users = repository.findAll();      // ③ interface type, not impl
        if (users.isEmpty()) {                        // ④ explicit guard
            return "id,name,email\n";                 //    header only — no null, no NPE
        }
        return users.stream()                         // ⑤ pipeline (Lesson 02)
            .filter(User::active)
            .map(User::toCsvRow)
            .collect(Collectors.joining("\n", "id,name,email\n", "\n"));
    }

    public String readExportFile(Path path) {         // ⑥ wraps checked → unchecked
        try (var reader = Files.newBufferedReader(path)) {   // ⑦ try-with-resources
            return reader.lines().collect(Collectors.joining("\n"));
        } catch (IOException e) {                     // ⑧ checked exception
            throw new ExportReadException(path, e);   // ⑨ translated into domain exception
        }
    }
}
```

- **②** — `String` return, never `null`; empty data is a valid result, not an absence.
- **③** — code against `List<User>`, not `ArrayList<User>`.
- **⑥** — `IOException` is checked, so it *must* be handled; the service wraps it into its own unchecked `ExportReadException` (carrying the original as `cause`).
- **⑦** — the reader is closed automatically even if reading throws mid-way.
- **⑨** — the caller (or the `@RestControllerAdvice`) sees one domain exception instead of a raw `IOException` leaking from the persistence/IO layer.

This is the real pattern you'll see in Spring codebases: **checked at the boundary, translated to domain exceptions, handled globally.**

## 7. Exercise — run it yourself (10 minutes)

```bash
jshell
```

```java
// 1) Erasure — what prints?
var a = new ArrayList<String>();
var b = new ArrayList<Integer>();
a.getClass() == b.getClass()
a.add("x");
String s = a.get(0);

// 2) Invariance — try each line, read the error on the failing ones
List<String> strings = List.of("a", "b");
List<Object> objs = strings;            // ❌ why?
List<? extends Object> any = strings;   // ✅ why does this work?
any.get(0)                              // ✅ read-only OK

// 3) Set order — what order prints for each?
new HashSet<>(List.of(3, 1, 2))
new LinkedHashSet<>(List.of(3, 1, 2))
new TreeSet<>(List.of(3, 1, 2))

// 4) Checked exceptions in jshell
// (jshell lets checked exceptions escape — note how it differs from a real file)
java.nio.file.Files.readString(java.nio.file.Path.of("nope.txt"))

// 5) Try-with-resources + custom exception
class ExportReadException extends RuntimeException {
    ExportReadException(java.nio.file.Path p, Exception cause) {
        super("cannot read " + p, cause);
    }
}
try (var r = java.nio.file.Files.newBufferedReader(java.nio.file.Path.of("nope.txt"))) {
    r.readLine();
} catch (java.io.IOException e) {
    throw new ExportReadException(java.nio.file.Path.of("nope.txt"), e);
}

/exit
```

**Report back what you see:**
- In (1): the two lists are the *same* class — but how can `s` still be a `String`?
- In (2): the exact wording of the invariance error — it's the same error you'll hit in real code within your first week.
- In (3): which implementation preserved insertion order? Which sorted?

## 8. Homework — generics + collections applied

In the `url-shortener/` project, add a small generic utility (new file `src/main/java/com/icd/urlshortener/common/Paging.java`):

```java
package com.icd.urlshortener.common;

public final class Paging {
    private Paging() {}   // utility class — no instances

    /** Returns up to `limit` items, or all if limit <= 0. Never null. */
    public static <T> List<T> firstN(List<T> items, int limit) {
        // your implementation: use streams + limit, handle null/empty gracefully
    }
}
```

Implementation notes: `items.stream().limit(limit).toList()` is the core; decide what `null` input means (throw `IllegalArgumentException`? return empty? — a deliberate design choice worth defending). Compile with `mvn -q compile` and report the result.

## Next

Lesson 04: **Spring Core — IoC/DI, beans, scopes, profiles**. You now have the whole Java delta (records, streams, Optional, generics, collections, exceptions). Time to wire beans and build the URL Shortener REST API (Phase 2–3).
