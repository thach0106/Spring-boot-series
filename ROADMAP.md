# Spring Boot Learning Roadmap (Java)

> Target: senior backend / system design interviews · Japanese enterprise market · Started 2026-08-07

## Goal & why

- Java + Spring Boot is the #1 backend stack in the **Japanese enterprise market** → widens the job search significantly
- ~70% of NestJS knowledge transfers (DI, ORM, controllers, validation, testing) — this is a dialect, not a new language
- One big mental shift: **threads instead of an event loop** (virtual threads since JDK 21)

## Status (updated 2026-08-11)

| Phase | Topic | Status | Notes |
|---|---|---|---|
| 0 | Environment setup | ✅ Done | JDK 21 Temurin · Maven 3.9.16 · Spring Boot 4.1.0 scaffold in `url-shortener/` (build + tests + health check verified) |
| 1 | Java language (delta from TS) | 🔄 In progress | Lessons 01–04 delivered (records, streams, Optional, generics, collections, exceptions, CompletableFuture, virtual threads) — Phase 1 deliverable (CSV export in Java) pending |
| 2 | Spring core (IoC/DI, beans, scopes, profiles, config) | 🔄 In progress | Lesson 05 delivered — Hello + 2 services DI exercise; typed-config homework pending |
| 3 | Web layer (MVC, validation, exceptions, Actuator) | 🔄 In progress | Lessons 06–07 delivered — URL Shortener REST API + JWT auth (register/login, filter chain, 401 entry point); pagination + slice test homework pending |
| 4 | Data (Spring Data JPA, Hibernate, Flyway, PostgreSQL) | ⬜ Not started | Deliverable: URL shortener + persistence |
| 5 | Integration (Redis cache, Kafka, scheduling, Resilience4j) | ⬜ Not started | Maps 1:1 onto News Feed design |
| 6 | Production (Testcontainers, metrics, Dockerfile, logging) | ⬜ Not started | Deliverable: runs in Docker with health checks |
| 7 | Optional: Spring Security, WebFlux/WebClient, Boot 4 / Java 25 diffs | ⬜ Not started | |

## Concept map: NestJS → Spring Boot

| Concept | NestJS / Node (known) | Java / Spring Boot (learning) |
|---|---|---|
| Language | TypeScript | Java 21 (LTS) |
| Package mgmt | npm / pnpm | Maven (or Gradle) |
| Scaffolding | `nest new` | Spring Initializr (`start.spring.io`) |
| Entry point | `main.ts` + `NestFactory.create` | `@SpringBootApplication` + `SpringApplication.run` |
| DI container | modules + `@Injectable` | IoC context + `@Component` / `@Service` |
| HTTP layer | `@Controller()` + `@Get()` | `@RestController` + `@GetMapping` |
| Validation | class-validator + `ValidationPipe` | Jakarta Validation `@Valid` |
| Error handling | `@Catch()` + ExceptionFilter | `@RestControllerAdvice` + `ProblemDetail` (RFC 7807) |
| Config | `@nestjs/config` + `.env` | `application.yml` + `@ConfigurationProperties` |
| ORM | TypeORM / Prisma | Hibernate via Spring Data JPA |
| Migrations | Prisma migrate / TypeORM | Flyway |
| Logging | pino / winston | SLF4J + Logback |
| Tests | Jest + supertest | JUnit 5 + Mockito + MockMvc |
| Messaging | `@nestjs/microservices` + Kafka | Spring Kafka (same Kafka) |
| Cache | cache-manager + Redis | Spring Cache `@Cacheable` |
| Scheduler | `@nestjs/schedule` | `@Scheduled` |
| OpenAPI | Swagger module | springdoc-openapi |
| Retry / circuit breaker | hand-rolled | Resilience4j |
| Metrics | Prometheus client | Micrometer + Actuator |
| Auth | Passport + JWT guards | Spring Security (hardest — last) |

## Where the transfer breaks (the real learning)

1. **Threads, not an event loop** — virtual threads make blocking cheap, but thread-safety (singletons shared) becomes your responsibility
2. **Annotations + proxies, not decorators** — `@Transactional`/`@Cacheable` work via proxies; `this.method()` calls silently bypass them
3. **Singleton by default** — beans are singletons; services must be stateless
4. **Auto-configuration magic** — starters + `@ConditionalOnClass`; learn to read the classpath (`mvn dependency:tree`)
5. **Lazy loading + N+1** — Hibernate lazy relations → `LazyInitializationException`, N+1 queries (same fixes as TypeORM: fetch joins, `@EntityGraph`, batch size)
6. **Version drift is real** — Spring Boot 4.x is current (2026); 3.5.x line ended at 3.5.16; Boot 4 renamed starters (`web` → `webmvc`) and split test starters

## Phases & deliverables

| Phase | Content | Deliverable |
|---|---|---|
| 1 (wk 1–2) | Java delta: records, streams, `Optional`, generics, collections, exceptions, `CompletableFuture`, virtual threads | Rewrite a small known thing (e.g. CSV export) idiomatically |
| 2 (wk 3) | Spring core: IoC/DI, bean scopes, profiles, `application.yml`, constructor injection | REST "Hello" + 2 services wired by DI |
| 3 (wk 4) | Web layer: `@RestController`, DTO records, `@Valid`, `@RestControllerAdvice`, pagination, Actuator | **URL Shortener REST API** |
| 4 (wk 5) | Data: Spring Data JPA, Flyway, PostgreSQL (Docker), `@Transactional`, N+1 fixing | URL shortener + persistence |
| 5 (wk 6) | Integration: Redis `@Cacheable`, Spring Kafka, `@Scheduled`, Resilience4j | Production-flavored URL shortener |
| 6 (wk 7) | Production: Testcontainers, metrics, Dockerfile, structured logging | Dockerized with health checks |
| 7 (opt.) | Spring Security (JWT), WebClient/WebFlux, Boot 4 / Java 25 | — |

## Production best practices (internalize as you go)

- Constructor injection — never field `@Autowired`
- Records for DTOs; never expose JPA entities in controllers
- Global errors via `@RestControllerAdvice` + `ProblemDetail`
- `@Transactional(readOnly = true)` for reads; short transactions; never in controllers
- Flyway migrations committed with code (like Prisma migrate)
- Typed config via `@ConfigurationProperties`; profiles per environment
- Stateless singleton services; state lives in DB/Redis
- Test pyramid: JUnit/Mockito unit → `@WebMvcTest`/`@DataJpaTest` slices → Testcontainers integration
- Actuator health/metrics on from day one

## Environment (rebuilt + verified live 2026-08-10)

- JDK: **Temurin 21.0.12+8 LTS** at `C:\Users\thach\dev\java\jdk-21.0.12+8` (user-level portable install; no admin needed). `JAVA_HOME` (User) points there; User var overrides the stale Machine `JAVA_HOME` (OpenJDK 11) and PATH `java` (JDK 15) — old installs still exist but are shadowed
- Maven: **3.9.16** at `C:\Users\thach\dev\maven\apache-maven-3.9.16`. MSYS fix applied: patched `bin/mvn` to add the missing **MinGW→Windows path conversion** (upstream only converts back for Cygwin → `ClassNotFoundException: classworlds.Launcher` on git-bash; the extra `if $mingw` block runs `cygpath --path --windows` on MAVEN_HOME/JAVA_HOME/CLASSWORLDS_JAR)
- Spring Boot: **4.1.0** (plain version on Central; Initializr still emits `4.1.0.RELEASE` which 404s — **patch pom.xml after every Initializr generation**). Starters renamed: `web`→`webmvc`, test starters split (`webmvc-test` etc.)
- Project: `url-shortener/` — Boot 4.1.0, Java 21, starters: `webmvc`, `validation`, `actuator` (+ `-test` variants). **`mvn test` → BUILD SUCCESS** (1 test, 2:02 min first run)
- Docker 29.3.1 (Docker Desktop; only `docker-desktop` WSL distro, no Ubuntu) — for PostgreSQL/Kafka in phases 4–6
- GitHub: `thach0106/Spring-boot-series`; repo-local git identity: `thach0106` / `thachnn@icd-vn.com` (global is `Thach Da` / `thachbovjp@gmail.com` — **repo-local override required, set via `git config user.name/email` inside the repo**)

## Lesson log

| Lesson | Topic | Date | Status |
|---|---|---|---|
| 01 | Java for NestJS devs: JVM mental model, syntax delta, records | 2026-08-07 | Delivered — JShell exercise pending |
| 02 | Streams + Optional: lazy pipelines, killing null | 2026-08-10 | Delivered — JShell exercise + CSV export homework pending |
| 03 | Generics, collections, exceptions: invariance, impl choices, checked vs unchecked | 2026-08-10 | Delivered — JShell exercise + Paging utility homework pending |
| 04 | Concurrency: threads, virtual threads, CompletableFuture — async without the event loop | 2026-08-11 | Delivered — JShell exercise + CSV export homework pending |
| 05 | Spring Core: IoC/DI, beans, scopes, profiles, typed configuration | 2026-08-11 | Delivered — DI exercise + AppProperties homework pending |
| 06 | Web layer: REST controllers, DTO records, Jakarta validation, ProblemDetail errors, Actuator | 2026-08-11 | Delivered — URL Shortener API exercise; pagination + slice test homework pending |
| 07 | Spring Security + AOP: JWT auth, filter chain, proxy mechanism, 401 vs 403 | 2026-08-11 | Delivered + **code verified** (compiled, ran, 8 curl probes passed) — roles homework pending |

## Lessons

- [01 — Java for NestJS developers](lessons/01-java-for-nestjs-developers.md)
- [02 — Streams + Optional](lessons/02-streams-and-optional.md)
- [03 — Generics, Collections, Exceptions](lessons/03-generics-collections-exceptions.md)
- [04 — Concurrency: CompletableFuture + virtual threads](lessons/04-concurrency-completablefuture-virtual-threads.md)
- [05 — Spring Core: IoC/DI, beans, profiles](lessons/05-spring-core-ioc-di-beans-profiles.md)
- [06 — Web layer: REST, validation, errors, Actuator](lessons/06-web-layer-rest-validation-errors-actuator.md)
- [07 — Spring Security + AOP: JWT auth](lessons/07-spring-security-jwt-aop.md)
