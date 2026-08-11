# Lesson 06 — Web layer: REST, validation, errors, Actuator

**Date:** 2026-08-11 · **Phase:** 3 (Web layer) · **Prereq:** Lesson 05 (Spring Core: DI, profiles)

## 1. The mental model that changes everything

You already know this layer from NestJS: controllers receive HTTP, DTOs validate, exception filters shape errors. The Spring equivalent uses the same ideas — the differences are *where* things live and *who* does the work:

1. **The framework calls YOU.** In NestJS you annotate handlers; in Spring, `@GetMapping` etc. map URLs to methods, and the framework invokes them on a request thread (Lesson 04 — that thread is your "async"). No `@Req()` plumbing — Spring binds arguments for you: path, query, body, headers.
2. **Validation happens BEFORE your method runs.** NestJS runs a `ValidationPipe` in the request pipeline. Spring runs Jakarta Validation *during argument binding* — if `@Valid` fails, your method is never called; the framework throws `MethodArgumentNotValidException` and the error goes to your advice. Same mental model as pipes, earlier in the flow.
3. **Errors are HTTP problems, not exceptions leaking out.** `ProblemDetail` (RFC 7807/9457) is the standard error body: `{type, title, status, detail, instance}`. NestJS has no built-in equivalent — you hand-roll error shapes; Spring gives you the industry standard out of the box.

The one architectural rule from the roadmap that this lesson makes concrete: **controllers are thin.** Parse → validate → call a service → return a DTO. No business logic in controllers — that's the service layer's job (your `UrlShortenerService`).

## 2. The syntax delta: NestJS → Spring

| NestJS | Spring |
|---|---|
| `@Controller('urls')` + `@Get(':id')` | `@RestController` + `@RequestMapping("/api/urls")` + `@GetMapping("/{id}")` |
| `@Param('id')` | `@PathVariable String id` |
| `@Query('page')` | `@RequestParam int page` |
| `@Body() dto: CreateDto` | `@Valid @RequestBody CreateUrlRequest request` |
| class-validator + `ValidationPipe` | Jakarta annotations + `@Valid` |
| `@Catch()` + `ExceptionFilter` | `@RestControllerAdvice` + `@ExceptionHandler` |
| `@HttpCode(201)` | `@ResponseStatus(HttpStatus.CREATED)` |
| `res.redirect()` | `ResponseEntity.status(302).location(...)` |
| `throw new NotFoundException()` | custom `RuntimeException` + advice mapping |
| `@nestjs/swagger` | springdoc-openapi (a later lesson) |

## 3. The controller — thin by design

```java
@RestController
@RequestMapping("/api/urls")
public class UrlController {
    private final UrlShortenerService urlShortenerService;

    public UrlController(UrlShortenerService urlShortenerService) {
        this.urlShortenerService = urlShortenerService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UrlResponse create(@Valid @RequestBody CreateUrlRequest request) {
        String id = urlShortenerService.create(request.originalUrl());
        return new UrlResponse(id, request.originalUrl(), "/" + id);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Void> resolve(@PathVariable String id) {
        String original = urlShortenerService.resolve(id)
            .orElseThrow(() -> new UrlNotFoundException(id));   // Optional → 404 (Lesson 02 pays off)
        return ResponseEntity.status(HttpStatus.FOUND)
            .location(URI.create(original))
            .build();
    }
}
```

Line by line:
- `@RestController` = `@Controller` + `@ResponseBody` — every return value is serialized to JSON (Jackson), no view templates.
- `@RequestMapping("/api/urls")` on the class = prefix for all mappings (like NestJS `@Controller('urls')`).
- `@RequestBody` — Jackson deserializes the JSON body into the record; `@Valid` triggers Jakarta validation before the method runs.
- `@PathVariable` — the `{id}` segment; Spring converts `String` automatically (and would convert `Long` too, with a 400 on bad input).
- Returning `ResponseEntity<Void>` with `302` + `Location` = an HTTP redirect — the *real* URL shortener behavior: a browser hitting the short URL lands on the original page.

**Why not return the entity/domain object?** Roadmap rule: *never expose internals.* The record DTO is the API contract; the service internals can change without breaking clients.

## 4. Validation — Jakarta, not class-validator

```java
public record CreateUrlRequest(
    @NotBlank(message = "originalUrl is required")
    @Pattern(regexp = "https?://.+", message = "must be a valid http(s) URL")
    String originalUrl,

    @Min(1)
    @Max(3650)
    Integer ttlDays
) {}
```

- Annotations live ON the record components — the DTO *is* the validation spec (vs NestJS where the class decorators sit beside the DTO).
- `@NotBlank` — not null and not whitespace. `@Pattern` — regex. `@Min`/`@Max` — numeric bounds.
- On failure, the framework throws `MethodArgumentNotValidException` and your method **never runs** — the pipe equivalent happens before invocation.
- Note: `Integer` (boxed), not `int` — so a missing `ttlDays` is `null`, not a 400. `int` would fail binding entirely. Boxed = optional, primitive = required.

## 5. Errors — `ProblemDetail` (RFC 7807)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UrlNotFoundException.class)
    public ProblemDetail handleNotFound(UrlNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("URL not found");
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .toList());
        return pd;
    }
}
```

- `@RestControllerAdvice` = global exception filter: *one* place, *every* controller (your `@Catch()` `ExceptionFilter`).
- `@ExceptionHandler(X.class)` — which exception maps to which response. The framework walks the advice list before the exception escapes to the client.
- `ProblemDetail` — the standard error body: `{"type":"about:blank","title":"URL not found","status":404,"detail":"..."}`. Machines can parse it; `pd.setProperty(...)` adds custom fields (here: the per-field validation errors).
- The custom exception is one line: `public class UrlNotFoundException extends RuntimeException { public UrlNotFoundException(String id) { super("URL not found: " + id); } }`

This is the roadmap's *global errors via `@RestControllerAdvice` + `ProblemDetail`* — and it's a top interview topic ("how do you handle errors consistently in Spring?").

## 6. Actuator — production observability from day one

Already on your classpath (`spring-boot-starter-actuator`). Default: only `/actuator/health` is exposed — and you saw it in Phase 0:

```bash
curl http://localhost:8080/actuator/health
# {"groups":["liveness","readiness"],"status":"UP"}
```

- `liveness` — "the app process is alive" (Kubernetes `livenessProbe`)
- `readiness` — "the app can serve traffic" (Kubernetes `readinessProbe`) — your k8s-lab knowledge plugs in directly
- Expose more in `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

Then `/actuator/info` (app metadata) and `/actuator/metrics` (JVM, HTTP counters) appear. In production you'd also secure them (Spring Security, Lesson 07) — never expose `metrics`/`env`/`beans` publicly.

## 7. Exercise — the URL Shortener API (Phase 3 deliverable, part 1)

In `url-shortener/`, **replace** `UrlShortenerService` and add four new files:

> **If you completed Lesson 05's exercise:** delete `web/HelloController.java` — it calls the old `shorten()` method, which no longer exists. `UrlController` supersedes it.

**(1) Replace `.../shortener/UrlShortenerService.java`** — state moves to a `ConcurrentHashMap` (Lesson 04: shared state must be explicitly thread-safe; the map IS the storage until Phase 4 gives us PostgreSQL):

```java
package com.icd.urlshortener.shortener;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import org.springframework.stereotype.Service;

@Service
public class UrlShortenerService {
    private final IdGenerator idGenerator;
    private final ConcurrentHashMap<String, String> store = new ConcurrentHashMap<>();

    public UrlShortenerService(IdGenerator idGenerator) {
        this.idGenerator = idGenerator;
    }

    public String create(String originalUrl) {
        String id = idGenerator.nextId();
        store.put(id, originalUrl);
        return id;
    }

    public Optional<String> resolve(String id) {
        return Optional.ofNullable(store.get(id));   // never null → Optional (Lesson 02)
    }
}
```

**(2) `.../web/CreateUrlRequest.java`** — request DTO with validation (section 4). **(3) `.../web/UrlResponse.java`** — `record UrlResponse(String id, String originalUrl, String shortUrl) {}`. **(4) `.../web/UrlNotFoundException.java`** — the one-line `RuntimeException` subclass. **(5) `.../web/GlobalExceptionHandler.java`** — section 5. **(6) `.../web/UrlController.java`** — section 3. Imports you'll need: `jakarta.validation.Valid` (for `@Valid`), `java.net.URI`, `org.springframework.http.HttpStatus` / `ResponseEntity`, `org.springframework.web.bind.annotation.*`.

**Run it and probe every case:**

```bash
mvn spring-boot:run

# 1. happy path — expect 201 + JSON
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"originalUrl":"https://example.com/long-page","ttlDays":30}' \
  http://localhost:8080/api/urls

# 2. invalid body — expect 400 + ProblemDetail with field errors
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"originalUrl":"not-a-url"}' \
  http://localhost:8080/api/urls

# 3. resolve — expect 302 with Location header (grab an id from #1)
curl -i http://localhost:8080/api/urls/<id-from-step-1>

# 4. unknown id — expect 404 + ProblemDetail
curl -i http://localhost:8080/api/urls/doesnotexist
```

**Report back:** the status codes and bodies of all four probes. In particular: what does the 400 body look like (the `errors` array)? What header does the 302 carry?

## 8. Homework — Phase 3 deliverable, part 2

1. **Wire `AppProperties`** (Lesson 05 homework) into `UrlResponse` so `shortUrl` is the full URL: `appProperties.baseUrl() + "/" + id`.
2. **Pagination** — roadmap Phase 3 item. Add `GET /api/urls` returning `PageResponse` (record: `items`, `page`, `size`, `total`) with `@RequestParam(defaultValue = "0") int page` and `@RequestParam(defaultValue = "10") int size` over the store's entries. Offset math: `entries.stream().skip(page * size).limit(size).toList()` — and note the flaw: `ConcurrentHashMap` iteration order is not insertion order (a preview of why Phase 4 needs a real database).
3. **Actuator**: add `management.endpoints.web.exposure.include: health,info,metrics` to `application.yml`, restart, and curl `/actuator/metrics` — find the JVM memory metric.
4. **First slice test** (preview of Phase 6): a `@WebMvcTest(UrlController.class)` with `MockMvc` — POST a valid and an invalid body, assert 201/400. This is your `supertest` equivalent; we'll formalize the test pyramid later.

## Next

Lesson 07: **Spring Security + AOP (`@Transactional` mechanics)** — JWT auth for the URL shortener, and the proxy mechanism that makes annotations work (the trap from Lesson 01's "where the transfer breaks").
