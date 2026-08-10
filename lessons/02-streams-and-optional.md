# Lesson 02 — Streams + Optional: pipelines and killing `null`

**Date:** 2026-08-10 · **Phase:** 1 (Java language) · **Prereq:** Lesson 01 (JVM, records)

## 1. The mental model that changes everything

In TypeScript you chain array methods directly on the array:

```typescript
const names = users.filter(u => u.age >= 18).map(u => u.name);
```

In Java the *shape* is the same — `filter` then `map` — but there is one crucial difference:

**TS array methods are eager. Java streams are lazy.**

```java
List<String> names = users.stream()
    .filter(u -> u.age() >= 18)
    .map(User::name)
    .toList();
```

**Mental image:** a stream is a *pipeline* — elements enter at the top, get transformed stage by stage, and only come out when a terminal operation (`.toList()`, `.count()`, `.sum()`) pulls them through. Before that terminal call, **nothing has run**. In TS, `.filter()` runs immediately and returns a new array; in Java, `.filter()` just *describes* a stage.

This is not a syntax difference — it's the whole design: lazy pipelines can short-circuit, skip work, and process elements one at a time instead of making intermediate copies.

## 2. The syntax delta, side by side

| TS (eager) | Java (lazy stream) |
|---|---|
| `arr.filter(fn)` | `arr.stream().filter(pred)` |
| `arr.map(fn)` | `arr.stream().map(fn)` |
| `arr.reduce((acc, x) => ..., init)` | `arr.stream().reduce(init, (acc, x) -> ...)` |
| `arr.find(fn)` | `arr.stream().filter(pred).findFirst()` → returns `Optional` |
| `arr.some(fn)` | `arr.stream().anyMatch(pred)` |
| `arr.every(fn)` | `arr.stream().allMatch(pred)` |
| `arr.flatMap(x => x.items)` | `arr.stream().flatMap(x -> x.items().stream())` |
| `arr.slice(0, 10)` | `arr.stream().limit(10)` |
| `arr.sort((a,b) => ...)` | `arr.stream().sorted(comparator)` |
| `arr.join(", ")` | `arr.stream().map(String::valueOf).collect(Collectors.joining(", "))` |
| `for (const x of arr)` | `arr.forEach(x -> ...)` (or enhanced for-loop) |
| `arr.findIndex(fn)` | `IntStream.range(0, arr.size()).filter(i -> ...).findFirst()` |

## 3. Streams — the three-part pipeline

Every stream expression has exactly **three parts**:

```java
users.stream()                          // ① SOURCE
     .filter(u -> u.age() >= 18)        // ② INTERMEDIATE ops (0..n) — lazy
     .map(User::name)                   //    another intermediate
     .toList();                         // ③ TERMINAL op — exactly one, triggers execution
```

- **① Source** — `collection.stream()`, `List.of(...).stream()`, or `IntStream.range(0, 10)`.
- **② Intermediate** — `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`. These return a *new stream* and **do nothing yet**. They're just adding stages to the pipeline.
- **③ Terminal** — `toList()`, `count()`, `sum()`, `reduce()`, `findFirst()`, `anyMatch()`. This is where elements actually flow. A stream without a terminal op is dead code — it never executes.

**Internal mechanism (why lazy matters):**

```
filter ──> map ──> toList
  u1        u1'      u1'  ← element 1 pulled through ALL stages
  u2        u2'      u2'  ← element 2, one at a time
  ...
```

Elements flow **vertically** (one element through all stages), not horizontally (all elements through one stage). Consequence:

- `findFirst()` stops the pipeline the moment one element survives — no full scan. In TS, `.filter().find()` scans everything *then* finds; in Java, filtering happens *during* the search and can stop early.
- `limit(5)` on an infinite stream (`Stream.iterate`) works — it pulls only 5.

**Production payoff:** you get streaming behavior over collections without writing loops, and short-circuiting comes free.

## 4. `Optional` — killing `null` the Java way

**Problem:** in TS, `user?.profile?.name ?? "unknown"` handles absence inline. Java has **no `?.` and no `??`** — every `null` dereference throws `NullPointerException`.

**Solution — `Optional<T>`:** a box that either *contains* a value or is *empty*. You can't get the value out without explicitly deciding what happens in the empty case.

```java
Optional<User> maybe = userService.findById(42L);

// TS:        userService.findById(42)?.name ?? "unknown"
String name = maybe.map(User::name).orElse("unknown");
```

**The three ways to create one:**

| Method | Meaning | TS equivalent |
|---|---|---|
| `Optional.of(x)` | x **must not** be null — throws `NPE` immediately if it is | — (fail fast) |
| `Optional.ofNullable(x)` | x may be null → empty box if so | `??` handling |
| `Optional.empty()` | explicitly empty | `undefined` |

**The three ways to read one:**

```java
maybe.orElse("fallback")          // value or fallback  →  ?? "fallback"
maybe.orElseGet(() -> compute())  // value or lazily-computed fallback (only runs if empty!)
maybe.orElseThrow(() -> new NotFoundException("user"))  // value or throw
```

**Chain without checking — this is the `?.` replacement:**

```java
// TS:  user?.profile?.address?.city ?? "unknown"
String city = Optional.ofNullable(user)
    .map(User::profile)          // empty if user is null  →  ?.
    .map(Profile::address)       // empty if profile is null  →  ?.
    .map(Address::city)          // empty if address is null  →  ?.
    .orElse("unknown");          // ?? "unknown"
```

**Production rules (the trade-offs):**

| DO | DON'T |
|---|---|
| Return `Optional<T>` from methods that may not produce a result (`findById`, lookups) | Never as a **field** — it's not serializable-friendly and adds noise |
| Use it in streams (`findFirst()`, `max()`, `reduce()` return it) | Never as a **method parameter** — forces callers into ceremony |
| Chain `.map()`/`.filter()` to transform while empty-safe | Never use it for **collections** — return an empty `List` instead of `Optional<List>` |
| `.orElseThrow()` with a domain exception in services | Don't over-opt: a method that *always* returns a value should return the plain type |

**Why `.orElseGet` over `.orElse`:** `orElse(x)` evaluates `x` **eagerly**, even when the box is full. If the fallback is expensive (`orElse(loadFromDb())`), you pay the DB call on every success. `orElseGet(() -> ...)` only runs when empty.

## 5. Reading your own code — streams + Optional in a real service

A URL-shortener snippet (Spring Service, Lesson 3 will unpack the annotations):

```java
@Service                                              // ① Spring bean
public class UrlStatsService {

    private final UrlRepository repository;           // ② injected dependency

    public UrlStatsService(UrlRepository repository) { // ③ constructor injection
        this.repository = repository;
    }

    public List<UrlStats> topClicked(int limit) {     // ④ returns a plain List
        return repository.findAll().stream()          // ⑤ source: all URLs
            .filter(Url::isActive)                    // ⑥ keep only active
            .sorted(Comparator.comparingLong(Url::clickCount).reversed()) // ⑦ highest first
            .limit(limit)                             // ⑧ take top N — short-circuits
            .map(UrlStats::from)                      // ⑨ entity → DTO record
            .toList();                                // ⑩ terminal: pull through
    }

    public Optional<String> originalUrl(String code) { // ⑪ declares: may be absent
        return repository.findByCode(code)            // ⑫ Optional<Url> from repo
            .filter(Url::isActive)                    // ⑬ empty if expired/disabled
            .map(Url::originalUrl);                   // ⑭ transform inside the box
    }
}
```

- **⑤** — `findAll().stream()`: source. (In production you'd paginate instead of loading all — Lesson 4.)
- **⑥→⑧** — all lazy; nothing scans until `toList()`.
- **⑨** — `UrlStats::from` is a static factory method reference (like passing a function).
- **⑪** — the *return type itself* tells callers "this may not exist". That's the contract.
- **⑫→⑭** — no `if (url != null)` anywhere. The box handles absence.

**Note the contrast:** `topClicked` returns an empty `List` when nothing matches (never `null`, never `Optional<List>`); `originalUrl` returns `Optional` because *a single missing value* is a legitimate outcome the caller must handle.

## 6. Exercise — run it yourself (10 minutes)

```bash
jshell
```

```java
// 1) What does each line print? (watch how laziness shows up)
var nums = List.of(1, 2, 3, 4, 5, 6);
nums.stream().filter(n -> n % 2 == 0).map(n -> n * 10).toList()
nums.stream().limit(3).toList()

// 2) findFirst short-circuits — count how many times the filter runs
var infinite = java.util.stream.Stream.iterate(0, n -> n + 1);
infinite.filter(n -> n % 7 == 0).findFirst().orElseThrow()

// 3) Optional basics
Optional.ofNullable(null).orElse("fallback")
Optional.ofNullable("x").map(String::toUpperCase).orElse("fallback")
Optional.<String>empty().orElseThrow(() -> new RuntimeException("missing"))
//    ^-- what does the <String> do? (type witness — javac can't infer it)

// 4) reduce — sum manually
nums.stream().reduce(0, (acc, n) -> acc + n)

/exit
```

**Report back what you see** — especially:
- In (2), does `findFirst` scan all numbers or stop early? How can you tell?
- In (3), what type does `orElseThrow` return, and when does the lambda actually run?

## 7. Homework — the Phase 1 deliverable

Rewrite something you know from NestJS idiomatically in Java: **CSV export**. Given a `List<User>` (record with `id`, `name`, `email`, `active`), produce a `List<String>` of CSV lines — only active users, columns `id,name,email`, header row first. Use `stream()`, `filter`, `map`, `Collectors.joining("\n")`. ~10 lines. Bring it next session.

## Next

Lesson 03: **generics + collections + exceptions** — the last syntax delta, then we wire beans with Spring Core (IoC/DI).
