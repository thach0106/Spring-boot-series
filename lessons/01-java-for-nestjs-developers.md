# Lesson 01 — Java for NestJS developers

**Date:** 2026-08-07 · **Phase:** 1 (Java language) · **Prereq:** none

## 1. The mental model that changes everything

JavaScript runs in a browser/Node. **Java runs on the JVM — a virtual machine that executes *bytecode*.**

| | TypeScript/Node | Java |
|---|---|---|
| Compile step | `tsc` → JS (type-check only) | `javac` → **bytecode** (mandatory, checked) |
| Execution | JS engine interprets + JITs | JVM loads bytecode, **JIT-compiles hot paths** |
| Platform | Node = the platform | **JVM = the platform** (same `.class` anywhere) |
| Types | erased at runtime | checked at compile time, on the class file |
| Objects | prototype chains | classes + interfaces |
| Concurrency | one thread, event loop | **thread-per-request** (+ virtual threads in 21) |
| Memory | V8 GC | JVM GC (young/old generations) |
| Dependencies | npm registry | Maven Central (`~/.m2`) |

Mental image: `.java` → `javac` → `.class` (bytecode) → packaged into a fat jar → JVM reads it, JIT-compiles hot paths into native code. **The bytecode is the contract, not your source.**

**New vocabulary:** **method** (function), **field** (property), **package** (namespace), **annotation** (`@X` — like decorators, but metadata Spring interprets at runtime), statements always end with `;`.

## 2. The syntax delta, side by side

```typescript
// TypeScript
interface User { id: number; name: string; }
const getUser = (id: number): User => ({ id, name: "Thach" });
```

```java
// Java
record User(long id, String name) { }
User getUser(long id) { return new User(id, "Thach"); }
```

| TS construct | Java equivalent |
|---|---|
| `const x = 5` | `int x = 5;` (type first, explicit) |
| `let s: string` | `String s;` |
| `interface` / `type` | `interface`, `record`, `class` |
| arrow `=>` | method body `{ ... }` |
| object literal `{ a: 1 }` | `new SomeClass(...)` or `record` |
| `null` / `undefined` | `null` only |
| template `` `${x}` `` | `"x=" + x` or `"%s".formatted(x)` |
| `import { x } from '...'` | `import com.icd.urlshortener.X;` |
| `class Foo { constructor(...) }` | `class Foo { Foo(...) }` (constructor = class name) |
| `async/await` | threads, `CompletableFuture` (Lesson 5) |

## 3. Records — first new tool

**Problem:** in TS a DTO is free (object literal). In old Java, a data class needed constructor + getters + equals + hashCode + toString (~50 lines).

**Solution — `record` (Java 16+):**

```java
record UrlRequest(String originalUrl, int ttlDays) { }
```

**Internal mechanism** (compiler generates):
- constructor `UrlRequest(String, int)`
- accessors `originalUrl()` / `ttlDays()` — no `get` prefix, no setters
- value-based `equals()` / `hashCode()` / `toString()`
- all fields `final` → **immutable by construction**

**Production use:** the idiomatic Spring Boot 4 DTO — controllers return them, Jackson serializes them to JSON with zero config. Your NestJS `CreateUrlDto` becomes a one-liner.

## 4. Reading your own code — line by line

`src/main/java/com/icd/urlshortener/UrlShortenerApplication.java`:

```java
package com.icd.urlshortener;                        // ① namespace; MUST match the folder path

import org.springframework.boot.SpringApplication;   // ② explicit import — no magic globals

@SpringBootApplication                               // ③ annotation: metadata Spring interprets at startup
public class UrlShortenerApplication {               // ④ public class; file name must equal class name

    public static void main(String[] args) {         // ⑤ JVM entry point — this exact signature
        SpringApplication.run(UrlShortenerApplication.class, args);   // ⑥ bootstrap: context + Tomcat
    }
}
```

- **① `package`** — like folder-based modules; `com.icd.urlshortener` maps to `src/main/java/com/icd/urlshortener/`
- **③ `@SpringBootApplication`** — one annotation = component scan + auto-configuration + configuration (decompressed in Lesson 3)
- **⑤ `public static void main`** — `static` = belongs to the class; the JVM calls it without creating an object
- **⑥** — `SpringApplication.run(...)` = `NestFactory.create(AppModule)` + `listen()` in one call: builds the context (your module graph), wires beans, starts embedded Tomcat

## 5. Exercise — run it yourself (10 minutes)

`jshell` is Java's REPL (ships with JDK 21). Open a terminal:

```bash
jshell
```

Type, one at a time, and **watch what each line prints**:

```java
record User(long id, String name) {}
var u = new User(1, "Thach");
u                    // ← what does jshell print? (toString)
u.name()             // ← accessor
u.name = "Lâm";      // ← what error? WHY?
"hello".repeat(3)    // ← Java strings are objects
var x = 5;           // ← var = type inference
/exit
```

The third line is the lesson: records are immutable — the compiler *refuses* mutation. Immutable DTOs are safe to share across threads (matters for the concurrency lesson).

**Report back what you see** (especially the error message — read it, it's informative).

## Next

Lesson 02: **streams + `Optional`** — `map/filter/reduce` on collections, and killing `null` bugs the Java way.
