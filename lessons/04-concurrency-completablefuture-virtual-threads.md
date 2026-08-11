# Lesson 04 — Concurrency: threads, virtual threads, CompletableFuture

**Date:** 2026-08-11 · **Phase:** 1 (Java language) · **Prereq:** Lesson 03 (generics, collections, exceptions)

## 1. The mental model that changes everything

In Node there is **one thread and an event loop**. You write `async/await` because you *must* — blocking the single thread stalls every request in the process.

In Java there are **many threads**. The JVM gives each request its own thread, so **blocking is normal and fine**. You write plain, synchronous code and the OS (platform threads) or the JVM (virtual threads) does the scheduling.

Three mental shifts:

1. **Concurrency ≠ async syntax.** `async/await` in Node is the *price* of having one thread. In Java it's the default model — which is exactly why Spring MVC controllers are plain methods: Tomcat hands each request a thread and calls your method. No `await` anywhere. (If a Node dev reads Spring code and thinks "where's the async?", the answer is: *the thread is the async*.)
2. **Virtual threads (JDK 21) give you the event loop's efficiency without its tax.** A platform thread costs ~1 MB of stack and OS scheduling — you can run a few thousand. A virtual thread costs a few KB and is scheduled by the JVM — you can run *millions*. When a virtual thread blocks on I/O, the JVM unmounts it and runs another on the same underlying thread. That is precisely the efficiency of `async/await` — but your code stays linear, no callback/promise splitting.
3. **`CompletableFuture` is Java's Promise** — reach for it when you genuinely need to *compose* parallel work (fan-out/fan-in, like your News Feed design). For ordinary request handling on a virtual thread, just block.

## 2. The syntax delta, side by side

| Node / TS | Java |
|---|---|
| single thread + event loop | platform threads + virtual threads |
| `async function f()` | plain method (thread-per-request) |
| `await promise` | `future.join()` |
| `Promise.all([a, b])` | `CompletableFuture.allOf(a, b).join()` |
| `p.then(v => ...)` | `future.thenApply(v -> ...)` |
| `p.then(v => anotherPromise)` | `future.thenCompose(v -> ...)` |
| `p.catch(e => ...)` | `future.exceptionally(e -> ...)` |
| `Promise.resolve(v)` | `CompletableFuture.completedFuture(v)` |
| `setTimeout(fn, ms)` | `Thread.sleep(ms)` |
| `Promise.allSettled([...])` | `allOf` + per-future `handle(...)` |

## 3. Platform vs virtual threads — the internals

**Platform thread** (what Java had until 21): 1 MB+ stack reserved, scheduled by the OS kernel, context switch is a kernel round-trip (microseconds). 10,000 platform threads ≈ 10 GB of virtual memory. This is *why* Node's event loop exists — OS threads were too expensive to scale.

**Virtual thread**: a small Java object until started; the stack grows on demand (a few KB idle). The JVM **mounts** it on a platform thread — the **carrier** — only while it actually runs. On a blocking call (socket read, `Thread.sleep`, DB call), the carrier is freed and immediately picks up another virtual thread.

**Mental image: carrier threads are lanes, virtual threads are cars.** Millions of cars, few lanes. A parked car (blocked on I/O) is pulled off the lane so other cars keep driving. The lane never idles.

```java
Thread.ofVirtual().name("worker-1").start(() -> doWork());   // one virtual thread
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {  // a pool of them
    executor.submit(() -> doWork());
}   // close() waits for all tasks — like Promise.all
```

Rules of thumb:
- **Blocking I/O on a virtual thread: fine** — that's what they're for.
- **Blocking I/O on a platform thread in a fixed pool: burns a scarce resource** — this is why old Java apps needed `async` libraries.
- **CPU-bound work**: use platform threads; parallel streams cap at `Runtime.getRuntime().availableProcessors()`.

## 4. Thread safety — why Spring beans must be stateless

Spring beans are **singletons**: ONE instance shared by *all* request threads. A mutable field means two concurrent requests race on it.

```java
// DANGER — two concurrent requests corrupt the counter
class CounterService {
    private long count;              // shared mutable state
    public void inc() { count++; }   // read-modify-write is NOT atomic
}
```

`count++` is three steps (read, add, write). Two threads can interleave: both read 5, both write 6 — one increment lost. This is a **race condition**, and it's the #1 bug class in naive Spring code.

Fixes, in order of preference:
1. **Immutability** — records, no mutable fields. The best fix is not needing one. (Remember Lesson 01: the compiler *refused* `u.name = "Lâm"` — that refusal protects you here.)
2. **Concurrent collections** — `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`.
3. **Atomics** — `AtomicLong`, `AtomicInteger` (CAS-based, lock-free).
4. **`synchronized` / locks** — last resort, short critical sections only.

## 5. CompletableFuture — Java's Promise

Use it for **parallel composition** (fan-out). The News Feed pattern from your system design practice maps directly:

```java
// Fan-out: fetch three sources in parallel, wait for all
CompletableFuture<List<String>> all = CompletableFuture
    .supplyAsync(() -> fetchTimeline(userId))       // task 1 on a worker thread
    .thenApply(timeline -> decorate(timeline))      // map: like .then()
    .exceptionally(err -> List.of())                // catch: fallback value
    .thenCompose(decorated -> fetchMentions(decorated));  // flatMap: like .then() with a promise

List<String> result = all.join();                   // await — blocks the calling thread
```

Line by line:
- `supplyAsync(...)` — runs the lambda on a worker pool, returns a `CompletableFuture` immediately (like calling an `async` function).
- `thenApply(fn)` — transform the value when ready (`p.then(v => ...)`).
- `exceptionally(fn)` — if anything upstream threw, produce a fallback (`p.catch(...)`).
- `thenCompose(fn)` — the function itself returns a future; flattens it (`thenApply` would give you `CompletableFuture<CompletableFuture<...>>` — the same trap as nested Promises).
- `join()` — await. On a virtual thread this is cheap; in Spring you rarely call it — the framework handles threads and you just return values.

Fan-out with `allOf` (`Promise.all`):

```java
CompletableFuture<?>[] tasks = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> fetch(url)))
    .toArray(CompletableFuture[]::new);
CompletableFuture.allOf(tasks).join();   // wait for all, discard results
```

## 6. Exercise — JShell (run it, watch the numbers)

```bash
jshell
```

**(1) Virtual threads make blocking cheap — 10 sleeps of 1s finish in ~1s:**

```java
var start = System.nanoTime();
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10)
        .forEach(i -> executor.submit(() -> { Thread.sleep(1000); return null; }));
}
System.out.println("10x1s sleeps took " + (System.nanoTime() - start) / 1_000_000 + " ms");
```

**(2) A race you can actually see:**

```java
import java.util.concurrent.atomic.AtomicLong;
var atomic = new AtomicLong();
IntStream.range(0, 10_000).parallel().forEach(i -> atomic.incrementAndGet());
System.out.println("atomic: " + atomic.get());          // 10000, always

long[] plain = {0};
IntStream.range(0, 10_000).parallel().forEach(i -> plain[0]++);
System.out.println("plain:  " + plain[0]);              // usually < 10000 — lost updates!
```

**(3) CompletableFuture chain:**

```java
var f = CompletableFuture.supplyAsync(() -> "hello")
    .thenApply(String::toUpperCase)
    .thenApply(s -> s + " world");
System.out.println(f.join());                           // HELLO world
```

**Report back what you see:**
- In (1): the elapsed ms — is it ~1000 or ~10000? Why?
- In (2): the two numbers. Run (2) a few times — does `plain` ever reach 10000? Why can `atomic` never lose an update?
- In (3): nothing to fix — just notice the shape: it reads like `Promise.resolve("hello").then(...).then(...)`.

## 7. Homework — Phase 1 deliverable: CSV export in Java

The roadmap's Phase 1 deliverable: *rewrite a small thing you know, idiomatically.* You've done CSV export in NestJS — now in Java, using everything from Lessons 01–04.

In `url-shortener/`, new file `src/main/java/com/icd/urlshortener/common/CsvExport.java`:

```java
package com.icd.urlshortener.common;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.function.Function;

public final class CsvExport {
    private CsvExport() {}

    /** Writes header + one row per item. Uses try-with-resources; escapes quotes. */
    public static <T> void export(Path file, List<String> header,
                                  List<T> rows, Function<T, List<String>> fields) throws IOException {
        // your implementation
    }
}
```

Requirements:
- Use **try-with-resources** (Lesson 03) and a stream pipeline (Lesson 02) for the rows
- Escape `"` and commas in field values (defensive — real CSV is unforgiving)
- **Bonus (virtual threads):** export 10 files in parallel using `Executors.newVirtualThreadPerTaskExecutor()` and time it against a sequential loop

Compile with `mvn -q compile` and report the result (and, for the bonus, the timing).

## Next

Lesson 05: **Spring Core — IoC/DI, beans, scopes, profiles**. Phase 1 is complete: records, streams, Optional, generics, collections, exceptions, concurrency. Time to wire beans and build the URL Shortener REST API (Phase 2–3).
