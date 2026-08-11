# Lesson 07 — Spring Security + AOP: JWT auth and the proxy mechanism

**Date:** 2026-08-11 · **Phase:** 7 (optional, pulled forward) · **Prereq:** Lesson 06 (web layer)

> **Why Security now?** The roadmap listed it as optional Phase 7, but the *proxy mechanism* behind `@Transactional`/`@Cacheable` is the roadmap's #2 "where the transfer breaks" item — and you'll need it to *understand* Phase 4 (Data). JWT auth also needs no database, so it slots in cleanly before PostgreSQL.

## 1. The mental model that changes everything

Two ideas, both of which look like magic until you see the mechanism:

**1. Annotations don't do anything. Proxies do.** `@Transactional`, `@Cacheable`, `@PreAuthorize` are *markers*. At startup, Spring sees the marker and wraps your bean in a **proxy** — a subclass (or interface implementation) that intercepts every method call, does the cross-cutting work (open transaction, check cache, check permission), and *then* delegates to your real method.

```
your code ──calls──▶ proxy ──before──▶ (open tx / check cache / check role)
                          │
                          └──▶ your real method
                          │
                     after ◀── (commit / cache result)
```

Consequence: **the proxy only applies when someone calls the bean *through* the container.** A call from inside the same class (`this.method()`) never passes through the proxy — the annotation silently does nothing. This is the trap, and we'll make it visible below.

**2. Spring Security is a chain of filters, not a magical wall.** An HTTP request walks through a `FilterChain` (like NestJS guards + interceptors, but as servlet filters): CORS → CSRF → authentication → authorization → … → *your controller*. Each filter either lets the request continue or stops it with 401/403. JWT auth = one custom filter that reads the `Authorization` header, validates the token, and stuffs the username into the `SecurityContext`.

## 2. The syntax delta: NestJS → Spring Security

| NestJS | Spring Security |
|---|---|
| `PassportModule` + `JwtModule.register(...)` | `spring-boot-starter-security` + `SecurityFilterChain` bean |
| `JwtStrategy.validate(payload)` | `OncePerRequestFilter` → sets `SecurityContextHolder` |
| `@UseGuards(JwtAuthGuard)` per controller | `.requestMatchers(...)` rules in one `SecurityConfig` |
| `@Roles('admin')` | `.hasRole("ADMIN")` / `@PreAuthorize("hasRole('ADMIN')")` |
| `bcrypt` via `bcryptjs` | `BCryptPasswordEncoder` (same algorithm) |
| `@Public()` decorator | `.permitAll()` on the path |
| exception filters for 401 | `AuthenticationEntryPoint` + `@RestControllerAdvice` |

## 3. The proxy mechanism — AOP in one page

### How `@Transactional` really works (the preview you'll need in Phase 4)

```java
@Service
public class UrlShortenerService {
    @Transactional
    public void create(String originalUrl) { ... }
}
```

1. At startup, a **post-processor** (from Lesson 05's construction sequence) sees `@Transactional`.
2. It creates a **CGLIB subclass** of your class — `UrlShortenerService$$SpringCGLIB$$0` — overriding every public method.
3. The bean stored in the container is that **proxy**, not your class. Your class exists only as the proxy's `super`.
4. When anything calls `create(...)`, it's the proxy's override that runs: open tx → call `super.create(...)` → commit/rollback.

### The trap: self-invocation

```java
@Service
public class UrlShortenerService {
    @Transactional
    public void createAndNotify() {
        create();       // ❌ this.create() — direct call, NO proxy, NO transaction!
    }

    @Transactional
    public void create() { ... }
}
```

`this.create()` is a plain Java call — the proxy is never involved. The annotation is dead code. Symptoms in production: "my transaction didn't roll back", "my cache wasn't invalidated".

**Fixes (in order):**
1. **Split the beans** — put `create()` in another service and inject it. The call crosses the container → proxy applies. This is the idiomatic fix.
2. Move the annotation to the *outer* method (`createAndNotify` gets `@Transactional`; the inner one is private or package-private).
3. Inject self (`ObjectProvider<UrlShortenerService>`, then `getObject()`) — works, but smells.

**How to *see* the proxy** (do this in the exercise):

```java
System.out.println(urlShortenerService.getClass().getName());
// com.icd.urlshortener.shortener.UrlShortenerService$$SpringCGLIB$$0
```

The `$$SpringCGLIB$$0` suffix is the proof: what you hold is a proxy. (This also explains Lesson 05's nuance: **records can't be `@Transactional`** — they're `final`, CGLIB can't subclass them. The container can still *create* records as beans — `@ConfigurationProperties` — but never *proxy* them.)

**Rollback rules (the interview favorite):** `@Transactional` rolls back on `RuntimeException`/`Error` by default — but **not on checked exceptions** (Lesson 03 pays off: `IOException` would *commit*). Fix: `@Transactional(rollbackFor = IOException.class)`. And `readOnly = true` on reads is a real optimization hint for the DB driver.

## 4. Spring Security: the filter chain + stateless JWT

**The chain** (simplified, in order):

```
request → CorsFilter → CsrfFilter → JwtAuthenticationFilter (ours)
        → AuthorizationFilter → ... → DispatcherServlet → your @RestController
```

**JWT flow (stateless — no server-side session):**
1. Client POSTs `/api/auth/login` with username/password → server verifies → returns a signed JWT.
2. Client sends `Authorization: Bearer <token>` on every call.
3. Our filter parses the token, checks the signature, puts the username in the `SecurityContext`.
4. The authorization filter compares the path against the rules in `SecurityConfig`.

**Why stateless?** The token is self-contained (claims + signature). Any server in the cluster can validate it without shared session storage — that's what makes horizontal scaling trivial. Trade-off: **you can't revoke a token server-side** (until expiry) — you need token blacklists or short TTLs (Lesson 5 topics).

## 5. Reading your own code — JWT auth on the URL shortener

### pom.xml — add the dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.7</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.7</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.7</version>
    <scope>runtime</scope>
</dependency>
```

> **Version note (the roadmap's "verify before trusting" habit):** `jjwt-api` 0.12.7 was the latest at time of writing — check Central (`repo1.maven.org/maven2/io/jsonwebtoken/jjwt-api/`) when you do this. The API changed between 0.11 (`parserBuilder()`) and 0.12 (`parser().verifyWith(...)`), so a newer major version may differ.

### `security/JwtService.java` — issue and verify tokens

```java
package com.icd.urlshortener.security;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import java.util.Date;
import javax.crypto.SecretKey;
import org.springframework.stereotype.Service;

@Service
public class JwtService {

    // DEV ONLY — in production this comes from config (Lesson 05's AppProperties), never source!
    private static final String SECRET = "url-shortener-dev-secret-change-me-0123456789";
    private static final long TTL_MS = 86_400_000; // 24h

    private final SecretKey key = Keys.hmacShaKeyFor(SECRET.getBytes());

    public String issue(String username) {
        return Jwts.builder()
            .subject(username)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + TTL_MS))
            .signWith(key)
            .compact();
    }

    public String parseUsername(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload()
            .getSubject();
    }
}
```

- `Keys.hmacShaKeyFor(...)` — HS256 needs a key ≥ 32 bytes; the secret is the *shared* secret, like a server-side API key.
- `issue(...)` — builds a JWT with claims (subject = username, issuedAt, expiration) and signs it.
- `parseUsername(...)` — verifies signature + expiry, returns the subject. Throws `JwtException` if tampered/expired.

### `security/JwtAuthenticationFilter.java` — the one filter that does the work

```java
package com.icd.urlshortener.security;

import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.List;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                String username = jwtService.parseUsername(header.substring(7));
                var authentication = new UsernamePasswordAuthenticationToken(username, null, List.of());
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (JwtException | IllegalArgumentException e) {
                SecurityContextHolder.clearContext(); // invalid token → treat as anonymous
            }
        }
        chain.doFilter(request, response);
    }
}
```

- `OncePerRequestFilter` — guaranteed to run once per request (servlet containers can re-dispatch).
- No token / invalid token → `SecurityContext` stays empty → the request continues as **anonymous** → the authorization filter decides (deny → 401/403).
- Valid token → we set the `Authentication` (username + empty authority list — roles come in homework) → the request continues *as that user*.

### `security/SecurityConfig.java` — the rules

```java
package com.icd.urlshortener.security;

import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                            // stateless API — no CSRF tokens
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()         // register/login are public
                .requestMatchers(HttpMethod.GET, "/api/urls/**").permitAll()  // redirects are public
                .anyRequest().authenticated())                       // everything else needs a JWT
            .exceptionHandling(ex -> ex.authenticationEntryPoint((request, response, authException) -> {
                // anonymous / invalid token → 401 + JSON, not Spring's default 403
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                response.getWriter().write(
                    "{\"status\":401,\"title\":\"Unauthorized\",\"detail\":\"Missing or invalid token\"}");
            }))
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

- `csrf.disable()` — CSRF protects *cookie-session* flows; a stateless bearer-token API doesn't need it (an attacker can't make the browser attach your JWT).
- `STATELESS` — no `JSESSIONID`, no server-side session. Every request is authenticated by its token alone.
- `requestMatchers(...)` — **rule order matters**: first match wins; specific rules before `anyRequest()`.
- `exceptionHandling(...)` — **the 403-vs-401 gotcha.** Spring's *default* for an anonymous user hitting a protected resource is **403 Forbidden** (as your probe will show if you remove this block — the `AuthenticationEntryPoint` defaults to denying). For a token API the correct status is **401 Unauthorized**, so we replace the entry point with one that writes a JSON body. NestJS guards return 401 by default; in Spring you must configure it explicitly.
- `addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)` — our filter runs *before* the auth machinery, so by the time authorization checks happen, the `SecurityContext` is populated.
- `BCryptPasswordEncoder` — the same bcrypt you know from NestJS; **never** store plaintext passwords.

### `security/UserStore.java` + `web/AuthController.java` — register/login

```java
package com.icd.urlshortener.security;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import org.springframework.stereotype.Service;

@Service
public class UserStore {

    // In-memory until Phase 4 (PostgreSQL). ConcurrentHashMap = Lesson 04 thread safety.
    private final ConcurrentHashMap<String, String> passwords = new ConcurrentHashMap<>();

    public void save(String username, String encodedPassword) {
        passwords.put(username, encodedPassword);
    }

    public Optional<String> findEncodedPassword(String username) {
        return Optional.ofNullable(passwords.get(username));
    }
}
```

```java
package com.icd.urlshortener.web;

import com.icd.urlshortener.security.InvalidCredentialsException;
import com.icd.urlshortener.security.JwtService;
import com.icd.urlshortener.security.UserStore;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final UserStore userStore;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;

    public AuthController(UserStore userStore, PasswordEncoder passwordEncoder, JwtService jwtService) {
        this.userStore = userStore;
        this.passwordEncoder = passwordEncoder;
        this.jwtService = jwtService;
    }

    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public AuthResponse register(@Valid @RequestBody RegisterRequest request) {
        userStore.save(request.username(), passwordEncoder.encode(request.password()));
        return new AuthResponse(jwtService.issue(request.username()), request.username());
    }

    @PostMapping("/login")
    public AuthResponse login(@Valid @RequestBody LoginRequest request) {
        String encoded = userStore.findEncodedPassword(request.username())
            .orElseThrow(InvalidCredentialsException::new);
        if (!passwordEncoder.matches(request.password(), encoded)) {
            throw new InvalidCredentialsException();
        }
        return new AuthResponse(jwtService.issue(request.username()), request.username());
    }
}
```

- **Never log passwords, never store them plaintext** — `passwordEncoder.encode(...)` at rest, `matches(...)` at login. The hash is what's stored; even a DB leak (Phase 4) doesn't leak passwords.
- `InvalidCredentialsException` — a one-line `RuntimeException`; the *same* exception for "user not found" and "wrong password" (don't leak which one — enumeration attack).

### DTO records + the 401 handler

```java
public record RegisterRequest(
    @NotBlank(message = "username is required")
    @Size(min = 3, max = 32, message = "username must be 3-32 chars")
    String username,

    @NotBlank(message = "password is required")
    @Size(min = 8, message = "password must be at least 8 chars")
    String password
) {}

public record LoginRequest(
    @NotBlank(message = "username is required")
    String username,
    @NotBlank(message = "password is required")
    String password
) {}

public record AuthResponse(String token, String username) {}
```

Add to `GlobalExceptionHandler` (Lesson 06) — the 401 case:

```java
@ExceptionHandler(InvalidCredentialsException.class)
public ProblemDetail handleInvalidCredentials(InvalidCredentialsException ex) {
    ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.UNAUTHORIZED, ex.getMessage());
    pd.setTitle("Unauthorized");
    return pd;
}
```

## 6. Exercise — secure the URL shortener, probe every case

Add the files above to `url-shortener/` (pom deps + `security/` 4 files + `web/AuthController`, `RegisterRequest`, `LoginRequest`, `AuthResponse`, `InvalidCredentialsException`, and the new `@ExceptionHandler`). Then:

```bash
mvn spring-boot:run
```

```bash
# 1. register — expect 201 + {"token":"eyJhbGciOi...","username":"thach"}
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"username":"thach","password":"password123"}' \
  http://localhost:8080/api/auth/register

# 2. login — expect 200 + token (save it: TOKEN=...)
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"username":"thach","password":"password123"}' \
  http://localhost:8080/api/auth/login

# 3. wrong password — expect 401 + ProblemDetail
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"username":"thach","password":"wrong"}' \
  http://localhost:8080/api/auth/login

# 4. weak password — expect 400 + validation errors array
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"username":"a","password":"x"}' \
  http://localhost:8080/api/auth/register

# 5. POST /api/urls WITHOUT token — expect 401 (Security denies, not your code)
curl -i -X POST -H "Content-Type: application/json" \
  -d '{"originalUrl":"https://example.com"}' \
  http://localhost:8080/api/urls

# 6. POST /api/urls WITH token — expect 201
curl -i -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"originalUrl":"https://example.com"}' \
  http://localhost:8080/api/urls

# 7. GET redirect stays public — expect 302 (works WITHOUT token — that's the design)
curl -i http://localhost:8080/api/urls/<id-from-step-6>
```

**See the proxy (2 minutes):** temporarily add to `UrlShortenerApplication`:

```java
@Bean
CommandLineRunner showProxy(UrlShortenerService s) {
    return args -> System.out.println("BEAN CLASS: " + s.getClass().getName());
}
```

Expected: `BEAN CLASS: com.icd.urlshortener.shortener.UrlShortenerService$$SpringCGLIB$$0` — the `$$SpringCGLIB$$0` is the proxy proof. Remove it after.

**See the 403 default (2 minutes, optional):** comment out the `.exceptionHandling(...)` block in `SecurityConfig`, restart, and re-run probe 5. You'll get **403** instead of 401 — that's Spring's default entry point. Uncomment after. This is the kind of "default that looks like a bug" that interviewers love.

**Report back:** the status codes for all 7 probes (especially 5: 401 with the JSON body — and what 403 looks like if you tried the optional step), and the `BEAN CLASS` line.

## 7. Homework

1. **Tamper test:** take the token from step 2, change one character in the middle, and curl step 6 with it. Expected: 401. Why? (The signature no longer matches — `parseUsername` threw.)
2. **Expiry:** set `TTL_MS = 2_000` (2 seconds), restart, login, wait 3s, reuse the token → 401. This is why clients re-login / refresh tokens.
3. **Roles (bonus):** add a `role` claim in `JwtService.issue(...)` (`claim("role", "ADMIN")`), read it in the filter into authorities (`List.of(new SimpleGrantedAuthority("ROLE_ADMIN"))`), then protect `POST /api/urls` with `.requestMatchers(HttpMethod.POST, "/api/urls/**").hasRole("ADMIN")` — and watch a registered user get 403.
4. **Security test (preview of Phase 6):** a `@WebMvcTest` with `@AutoConfigureMockMvc(addFilters = false)` — skip the filter chain and test the controller; or `@SpringBootTest` + real filter chain for a full-stack auth test. Both are the `supertest` equivalents.
5. **Think about production:** where should `SECRET` live? (Lesson 05 answer: `AppProperties` + env var / secret manager, never in source.) What's the operational cost of a 24h TTL? (A leaked token is valid for 24h — trade-off between UX and security.)

## Next

Lesson 08: **Spring Data JPA + Flyway + PostgreSQL** (Phase 4) — replace `ConcurrentHashMap` and `UserStore` with a real database, now that you understand the `@Transactional` proxy underneath it. The N+1 problem and `@EntityGraph` are on the menu.
