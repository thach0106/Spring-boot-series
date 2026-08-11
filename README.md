# Spring-boot-series

Java + Spring Boot learning series for a NestJS/Node backend engineer — targeting senior backend & system design interviews and the Japanese enterprise market.

## Contents

- [ROADMAP.md](ROADMAP.md) — phases, NestJS→Spring concept map, environment notes
- [lessons/](lessons/) — lesson notes, in order:

| # | Lesson | Topics |
|---|---|---|
| 01 | [Java for NestJS developers](lessons/01-java-for-nestjs-developers.md) | JVM mental model, syntax delta, records |
| 02 | [Streams + Optional](lessons/02-streams-and-optional.md) | Lazy pipelines, filter/map/reduce, killing null |
| 03 | [Generics, Collections, Exceptions](lessons/03-generics-collections-exceptions.md) | Invariance, implementation choices, checked vs unchecked |
| 04 | [Concurrency: CompletableFuture + virtual threads](lessons/04-concurrency-completablefuture-virtual-threads.md) | Threads vs event loop, virtual threads, Promise equivalents |
| 05 | [Spring Core: IoC/DI, beans, profiles](lessons/05-spring-core-ioc-di-beans-profiles.md) | ApplicationContext, constructor injection, scopes, typed config |
| 06 | [Web layer: REST, validation, errors, Actuator](lessons/06-web-layer-rest-validation-errors-actuator.md) | @RestController, Jakarta validation, ProblemDetail, probes |
| 07 | [Spring Security + AOP: JWT auth](lessons/07-spring-security-jwt-aop.md) | Filter chain, proxy mechanism, 401 vs 403, roles |
| 08 | [Spring Data JPA + Flyway + PostgreSQL](lessons/08-spring-data-jpa-flyway-postgresql.md) | Entities, repositories, N+1, transactions |

## Repo layout

- `url-shortener/` — Spring Boot 4.1.0 (Java 21) scaffold, built lesson-by-lesson into a production-flavored URL shortener REST API
