# Java 17 → Java 21 (LTS) Migration Notes

This document describes the migration of the Commerzbank-Demo banking microservices from
**Java 17** to **Java 21 (LTS)**.

## Summary

The codebase already targeted Java 17 (there was no Java 11 in the build). The migration
raises the language/runtime target to **Java 21 LTS** across all seven Maven modules and
aligns the Spring Boot patch version to gain official Java 21 runtime support, with no
business-logic or API changes.

## What changed

| Area | Before | After | Files |
|---|---|---|---|
| Java language/runtime target | 17 | **21** | `*/pom.xml` (`<java.version>`) |
| Spring Boot version | 2.7.14 / 2.7.15 (mixed) | **2.7.18** (standardized) | `*/pom.xml` (parent `<version>`) |
| Container base image | `openjdk:17-jdk` | **`eclipse-temurin:21-jdk-jammy`** | `User-Service/Dockerfile` |
| Docs | "Java 17" | "Java 21 (LTS)" + this guide | `README.md`, `docs/JAVA_21_MIGRATION.md` |

Modules updated (all 7): `Service-Registry`, `API-Gateway`, `User-Service`,
`Account-Service`, `Transaction-Service`, `Fund-Transfer`, `Sequence-Generator`.

Spring Cloud remains **2021.0.8** (the correct, compatible train for Spring Boot 2.7.x).

## Why it changed

- **`<java.version>21`** sets the `maven.compiler.release` (via the Spring Boot parent) so
  sources are compiled and run against the Java 21 bytecode level (class file v65).
- **Spring Boot 2.7.18** is the key compatibility upgrade. Java 21 runtime support requires
  **Spring Framework 5.3.30+**; Boot 2.7.18 ships Spring Framework **5.3.31** and bumps
  **Lombok to 1.18.30** (the first Lombok line that supports Java 21 annotation processing).
  Boot 2.7.14/2.7.15 ship older Spring Framework/Lombok that fail or are unsupported on
  Java 21. Standardizing all modules on 2.7.18 also removes the pre-existing version drift
  (some modules were 2.7.14, others 2.7.15).
- **Temurin 21 base image** replaces the now-deprecated `openjdk` Docker Hub image with a
  maintained, LTS-supported distribution at the Java 21 level.

This satisfies the constraint of upgrading **only what Java 21 requires** — no functional
dependencies (Keycloak admin client 21.0.1, ModelMapper 3.1.1, MapStruct/MySQL connector,
etc.) were changed.

## Compatibility risks & how they were handled

- **Lombok annotation processing on Java 21** — resolved by Boot 2.7.18 → Lombok 1.18.30.
  Verified: all modules compile cleanly.
- **ByteBuddy/Mockito on Java 21** — the test suite uses only `@SpringBootTest` context-load
  tests (no Mockito mocking), so ByteBuddy is not exercised; tests pass without overriding
  ByteBuddy. *If unit tests that use `@MockBean`/Mockito are added later, override
  `byte-buddy.version` to ≥ 1.14.x (and consider Mockito 5.x).* See "Follow-up".
- **Removed/legacy JVM options** — none are set in the build or run scripts, so nothing had
  to be removed. (No `--illegal-access`, CMS GC flags, etc. are used.)
- **Reflection / strong encapsulation (JPMS)** — the apps are non-modular (no
  `module-info.java`); Spring/Hibernate proxying uses Spring's repackaged CGLIB which is
  Java 21-aware in Spring Framework 5.3.31. No `--add-opens` were needed at runtime.
- **Staying on Spring Boot 2.7.x (vs 3.x)** — 2.7.x is in commercial/extended support, not
  OSS OGA. Java 21 *runs* on 2.7.18, but the strategic target remains Spring Boot 3.2+ for
  long-term Java 21 first-class support (separate, larger effort: `javax`→`jakarta`).

## Downstream impact for Commerzbank product teams

- **Build/CI agents and developer machines must provide a JDK 21 toolchain.** Anything
  pinned to JDK 17 will fail to compile (release 21 bytecode).
- **Runtime/base images**: deployment platforms must run on a Java 21 JRE/JDK. The
  `User-Service` image now requires `eclipse-temurin:21`. Other services are launched from
  the executable Spring Boot jar and need a Java 21 runtime on the host/image.
- **No API/contract changes**: REST endpoints, request/response DTOs, Eureka service IDs,
  gateway routes, and Keycloak integration are unchanged — consumers need no code changes.
- **Behavioral parity**: default GC on Java 21 is G1 (same family as 17); no GC/flags
  changes were introduced. Teams should still re-baseline performance in a staging
  environment before production rollout.

## Rollback considerations

- The change is **isolated to `pom.xml` versions, one Dockerfile line, and docs**, so
  rollback is reverting this PR/commit (restores Java 17 + Boot 2.7.14/2.7.15 + `openjdk:17`).
- No database schema, message contract, or API change is involved, so rollback is safe and
  does not require data migration.
- Build agents/images would need to revert to a JDK 17 toolchain to rebuild the old version.
- Because services are independently deployable, rollback can be done per-service if needed
  (all remain wire-compatible during a mixed 17/21 rollout).

## Validation performed (on JDK 21.0.11, Temurin)

- **Clean build**: `mvn clean install` succeeded for all 7 modules.
- **Unit tests**: 3 `@SpringBootTest` context-load tests (Service-Registry, API-Gateway,
  Account-Service) — **all pass**. Other 4 modules have no tests.
- **Integration tests**: none exist in the repo (documented gap, see Follow-up).
- **Application startup**: Eureka + all 6 services start on Java 21 and register UP in Eureka.
- **End-to-end smoke**: JWT obtained from Keycloak; `POST /api/users/register` and
  `GET /api/users` through the API Gateway return `200` (user persisted in DB + Keycloak).
- **Container build**: `User-Service` image builds; `java -version` inside the image reports
  **Temurin 21.0.11 LTS**.

## Assumptions

- Target is the latest Java 21 LTS toolchain (verified with Temurin 21.0.11).
- The intent is to remain on Spring Boot 2.7.x for this migration (minimal, focused change),
  not to perform the Spring Boot 3.x upgrade in the same change set.
- Keycloak and MySQL remain the external dependencies; their versions are out of scope.

## Unresolved risks / follow-up recommendations

- **No CI pipeline exists** in this repo (no `.github/`, GitLab CI, or Jenkinsfile). Add one
  that builds/tests on JDK 21 to guard the toolchain going forward.
- **Very low test coverage** (3 trivial context-load tests). Add real unit/integration tests
  (e.g. Testcontainers) — and when Mockito-based tests are introduced, pin
  `byte-buddy.version` ≥ 1.14.x for Java 21.
- **Strategic upgrade to Spring Boot 3.2+** for first-class Java 21 support and continued OSS
  maintenance (involves `javax`→`jakarta` and Keycloak adapter changes) — track separately.
- Only `User-Service` has a Dockerfile; if the other services are containerized, apply the
  same Temurin 21 base image.
