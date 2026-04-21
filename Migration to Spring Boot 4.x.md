# Migration to Spring Boot 4.x

## Current State

- **Starter** (`shop-spring-boot-starter:1.1.19`) is on Spring Boot **3.5.6**, Java 17
- **Spring Cloud**: `2025.0.1` (Eureka `4.3.1`, OpenFeign `4.3.1`)
- **springdoc-openapi**: `2.8.0`
- **All services** inherit the starter as their `<parent>`, receiving Spring Boot transitively
- Services are on different starter versions (`1.1.13`–`1.1.19`), so older services would need to bump the starter version to pick up the Spring Boot 4.x upgrade

---

## Breaking Changes in Spring Boot 4.x

| Area | Change | Impact |
|---|---|---|
| **Java** | Minimum **Java 21** (Java 17 dropped) | Starter and every service must upgrade `java.version` and `maven.compiler.release` to `21` |
| **Spring Framework** | Bumps to **7.x** | Removed APIs deprecated in SF5/SF6 — potential compilation errors across services |
| **Spring Cloud** | Requires new release train (`2026.x`) | `spring-cloud.version`, `spring-cloud-eurika.version`, `spring-cloud-openfeign.version` all need updating |
| **springdoc-openapi** | `2.x` may not support SB4; likely requires `3.x` | Used in Gate, Order, Product modules — verify compatibility |
| **Keycloak** | `keycloak-admin-client` must be compatible with Spring Security 7 | Gate uses `26.0.7` — needs verification against SB4 / Spring Security 7 |

---

## Required Changes

### In the Starter (`pom.xml`)

1. Bump `spring-boot-starter-parent` to the target `4.x` version
2. Change `java.version` and `maven.compiler.release` from `17` to `21`
3. Update `spring-cloud.version` to the compatible `2026.x` release train
4. Update `spring-cloud-eurika.version` and `spring-cloud-openfeign.version` to compatible versions
5. Update `springdoc-openapi.version` to `3.x` (once stable and SB4-compatible)
6. Publish as a new **major version** (e.g., `2.0.0`) so services can opt in gradually

### In Every Service

1. Bump the starter `<parent>` version to `2.0.0`
2. Override `java.version` to `21` if set at service level
3. Fix any compilation errors caused by removed Spring Framework 7 APIs
4. Update Docker base images to use a JDK 21 runtime
5. Update CI/CD pipelines to use Java 21

### Service-Specific Checks

| Service | Extra concern |
|---|---|
| **Gate** | Verify `keycloak-admin-client:26.0.7` and `nimbus-jose-jwt:10.5` compatibility with Spring Security 7 |
| **Gate** | WebFlux + WebMVC dual usage — review any changes in reactive stack under SB4 |
| **Config-server** | `spring-cloud-config-server` must align with the new Spring Cloud release train |
| **Eureka-server** | `spring-cloud-starter-netflix-eureka-server` must align with the new Spring Cloud release train |
| **Order, Product, Stock, etc.** | Check `spring-kafka` version managed by SB4 for compatibility |

---

## Recommended Migration Strategy

### Phase 1 — Prepare
- Upgrade all services to the latest `1.x` starter version first (unify on `1.1.19`)
- Audit usage of any deprecated Spring Framework 5/6 APIs across all services
- Verify that `springdoc-openapi 3.x`, the new Spring Cloud release train, and `keycloak-admin-client` all have stable SB4-compatible releases

### Phase 2 — Migrate the Starter
- Create a new branch in the starter repo
- Apply all dependency changes listed above
- Publish as `2.0.0-SNAPSHOT` for testing

### Phase 3 — Canary Service
- Pick a low-risk service (e.g., `Config-server` or `Eureka-server`) as the first to adopt `2.0.0`
- Upgrade Java runtime in its Docker image and CI pipeline
- Verify it builds, starts, and passes tests

### Phase 4 — Roll Out
- Migrate remaining services one by one, starting with infrastructure services (`Eureka`, `Config`) before business services (`Order`, `Product`, `Gate`, etc.)
- Address compilation and runtime issues per-service before moving to the next
- Publish starter `2.0.0` stable only after at least one full end-to-end test with all services

---

## Key Risk: Java 17 → 21

The Java version bump is the hardest constraint — it affects:
- All **Docker base images** (must switch to JDK 21 variants)
- **CI/CD pipelines** (GitHub Actions, Jenkins, etc. must install Java 21)
- Any **external tooling** (code analysis, agents) tied to a specific JDK version

Consider enabling Java 21 **virtual threads** (Project Loom) once migrated — Spring Boot 4.x supports them out of the box and can improve throughput in IO-bound services.