# Lesson 05 — Spring Core: IoC/DI, beans, scopes, profiles

**Date:** 2026-08-11 · **Phase:** 2 (Spring core) · **Prereq:** Lesson 04 (concurrency), Phase 0 scaffold (`url-shortener/`)

## 1. The mental model that changes everything

You already know DI from NestJS: modules declare providers, the container injects them. Spring's **IoC container** is the same idea with different vocabulary — and three real differences:

1. **The container owns everything.** You never `new` a bean. The **ApplicationContext** is your module graph: it scans for classes, builds beans, resolves dependencies, and destroys them on shutdown. Your code only *declares* and *receives*.
2. **Component scan replaces module wiring.** In NestJS you must register every provider in a `@Module` array. In Spring, `@SpringBootApplication` scans the whole package tree and picks up anything annotated `@Component`/`@Service`/`@Repository`/`@Controller` automatically. Fewer files to touch — but less explicit; you must know *what's on the classpath* (this is where the "magic" lives).
3. **Singleton by default — and that's the point.** One instance per context, shared by every thread. This is exactly why Lesson 04's statelessness rule matters: a singleton with a mutable field is a race condition waiting to happen.

Bonus mental model: **fail fast.** If a dependency is missing or circular, the app *refuses to start* with a clear error ("No qualifying bean of type ..."). In NestJS you often discover a broken graph at runtime; Spring discovers it at boot. That's the container's contract: **either the whole graph is valid, or nothing runs.**

## 2. The syntax delta: NestJS → Spring

| NestJS | Spring |
|---|---|
| `@Module({ providers: [...] })` | component scan, or `@Configuration` + `@Bean` methods |
| `@Injectable()` | `@Service` / `@Component` / `@Repository` / `@Controller` |
| `constructor(private svc: XService)` | `public XService(XService svc)` — constructor injection, no annotation needed |
| `AppModule` + `NestFactory.create` | `@SpringBootApplication` + `SpringApplication.run` |
| providers registered by hand | automatic component scan |
| custom provider / `useFactory` | `@Bean` method inside a `@Configuration` class |
| `@Inject(TOKEN)` | `@Qualifier("name")` when several beans share a type |
| `NODE_ENV` + `.env` | `application.yml` + `spring.profiles.active` |
| `ConfigService.get('KEY')` | `@Value("${key}")` or typed `@ConfigurationProperties` |

## 3. Beans — what they are and how the context builds them

A **bean** is an object whose lifecycle the container manages. Three ways to declare one:

1. **Stereotype annotations** — `@Service`, `@Component`, `@Repository`, `@Controller` (picked up by component scan). This is 90% of your code.
2. **`@Bean` methods** — inside a `@Configuration` class, for classes you can't annotate (third-party libraries, `RestClient`, `ObjectMapper`). The NestJS equivalent is a custom provider factory.
3. **Auto-configuration** — Spring Boot's magic layer; decompressed in Lesson 06.

**Construction sequence** (the internal mechanism):
1. Scan the classpath → find candidate classes
2. Order by dependency: resolve each bean's constructor parameters
3. Instantiate singletons → run post-processors (this is where AOP proxies get applied — the mechanism behind `@Transactional`, Lesson 07)
4. **Fail fast** — missing bean, ambiguous bean, or cycle → startup aborts with a precise error

`@Service` vs `@Component`? Semantically identical (`@Service` is meta-annotated with `@Component`). The distinction is *intent*: `@Service` = business logic, `@Repository` = data access (and adds exception translation once Spring Data is on the classpath), `@Controller` = web layer. Classic interview question — the answer is "they're all components; the names document the layer."

Default bean name = class name decapitalized: `UrlShortenerService` → `urlShortenerService`.

## 4. Constructor injection — the one rule to live by

```java
@Service
public class UrlShortenerService {
    private final IdGenerator idGenerator;              // final — can't be swapped later

    public UrlShortenerService(IdGenerator idGenerator) {  // constructor injection
        this.idGenerator = idGenerator;
    }
}
```

Why this is THE Spring best practice (roadmap rule #1):

- **Fully constructed or not at all** — no `null` fields, no half-built objects. This is immutability (Lesson 01) applied to beans.
- **Testable without a mocking framework**: `new UrlShortenerService(new FakeIdGenerator())` in a unit test. No reflection, no mocks.
- **Explicit**: the dependency list is the constructor signature — anyone can see what the bean needs.

Anti-patterns to know (for interviews — "why not field injection?"):
- `@Autowired private IdGenerator idGenerator;` — **field injection**: hides dependencies, breaks `final`, needs reflection to test. Never in production code.
- Setter injection — exists for optional/mutable deps; rare.

Production note: teams often use Lombok's `@RequiredArgsConstructor` to generate the constructor. Fine. Records can't be beans (the container needs mutable proxies) — **beans are classes, DTOs are records.** Keep that split.

## 5. Scopes — and why singleton is the default

| Scope | Lifecycle | Use |
|---|---|---|
| `singleton` (default) | one instance per context | stateless services — ~99% of beans |
| `prototype` | new instance per lookup/injection | stateful helpers (rare) |
| `request` | one per HTTP request | request-scoped holders |
| `session` | one per HTTP session | legacy web app state |

**The classic trap:** injecting a `prototype` bean into a `singleton` — the singleton grabs ONE prototype instance at construction and keeps it forever. If you genuinely need a fresh instance per call, inject `ObjectProvider<Thing>` and call `.getObject()`.

## 6. Profiles + typed configuration — the `.env` replacement

`src/main/resources/application.yml` — one file, whole config:

```yaml
spring:
  profiles:
    active: dev
```

```yaml
# application-dev.yml — profile-specific overrides
app:
  base-url: http://localhost:8080
  ttl-days: 30
```

```yaml
# application-prod.yml
app:
  base-url: https://short.example.com
  ttl-days: 365
```

Profiles are Spring's `NODE_ENV`. Activate via `spring.profiles.active`, the `SPRING_PROFILES_ACTIVE` env var, or `--spring.profiles.active=prod` on the command line.

**Typed config — the production way** (instead of scattered `@Value("${app.base-url}")`):

```java
@ConfigurationProperties(prefix = "app")
public record AppProperties(String baseUrl, int ttlDays) { }
```

- Binds `app.*` keys from YAML into a typed record — compile-time checked, like NestJS `ConfigModule` + validation but with the compiler on your side
- Kebab-case keys relax to camelCase: `base-url` → `baseUrl`
- Register once with `@ConfigurationPropertiesScan` on the application class

This is the roadmap rule: *typed config via `@ConfigurationProperties`, profiles per environment.*

## 7. Exercise — wire the first beans in the real project

Type these into `url-shortener/` (three new files):

**(1) `src/main/java/com/icd/urlshortener/shortener/IdGenerator.java`** — a service with no dependencies:

```java
package com.icd.urlshortener.shortener;

import org.springframework.stereotype.Service;

@Service
public class IdGenerator {
    private static final String ALPHABET =
        "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public String nextId() {
        return Long.toString(System.currentTimeMillis(), 36)
             + ALPHABET.charAt((int) (Math.random() * ALPHABET.length()));
    }
}
```

**(2) `.../shortener/UrlShortenerService.java`** — depends on `IdGenerator` via constructor injection:

```java
package com.icd.urlshortener.shortener;

import org.springframework.stereotype.Service;

@Service
public class UrlShortenerService {
    private final IdGenerator idGenerator;

    public UrlShortenerService(IdGenerator idGenerator) {
        this.idGenerator = idGenerator;
    }

    public String shorten(String originalUrl) {
        return "/" + idGenerator.nextId();
    }
}
```

**(3) `.../web/HelloController.java`** — a preview of Phase 3 (full MVC comes next lesson); just enough to *see* the wiring work. Note the record as response DTO → Jackson turns it into JSON automatically:

```java
package com.icd.urlshortener.web;

import com.icd.urlshortener.shortener.UrlShortenerService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    private final UrlShortenerService urlShortenerService;

    public HelloController(UrlShortenerService urlShortenerService) {
        this.urlShortenerService = urlShortenerService;
    }

    public record HelloResponse(String message, String shortUrl) {}

    @GetMapping("/api/hello")
    public HelloResponse hello() {
        return new HelloResponse("hello from Spring", urlShortenerService.shorten("https://example.com"));
    }
}
```

**Run it:**

```bash
mvn spring-boot:run        # terminal 1 — watch the log: Tomcat on 8080, beans created
curl http://localhost:8080/api/hello    # terminal 2
```

Expected: `{"message":"hello from Spring","shortUrl":"/m5x2a1b"}` — the container built `UrlShortenerService`, injected `IdGenerator`, and a fresh id appears on every call.

**Profile demo** — add `application-prod.yml` with `server.port: 8081`, then:

```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--spring.profiles.active=prod
curl http://localhost:8081/api/hello
```

Same endpoint, different port — the profile changed the environment.

**Report back:** the curl output from both runs, and the first log line that shows the active profile (it says `No active profile set` or `The following 1 profile is active: "prod"`).

## 8. Homework — Phase 2 deliverable: typed config + DI in the real project

1. Add `AppProperties` (record + `@ConfigurationProperties(prefix = "app")`) and put `@ConfigurationPropertiesScan` on `UrlShortenerApplication`. Bind `base-url` and `ttl-days`.
2. Change `UrlShortenerService.shorten(...)` to return `appProperties.baseUrl() + "/" + id` (inject `AppProperties` via the constructor — now it has TWO constructor params; the container resolves both).
3. `application-prod.yml`: `app.base-url: https://short.example.com` + `server.port: 8081`. Run both profiles, curl both, and confirm the shortUrl changes between dev and prod.
4. Preview of Lesson 06 (auto-configuration): run `mvn -q dependency:tree | head -40` and find where **Tomcat**, **Jackson**, and **Spring MVC** come from (which starter pulled them in?).

Compile with `mvn -q compile` as you go, run both profiles, and report the two curl outputs.

## Next

Lesson 06: **Web layer — `@RestController`, DTO records, `@Valid`, `@RestControllerAdvice`, Actuator** (Phase 3: the URL Shortener REST API). The container is now your mental model — time to put HTTP on top of it.
