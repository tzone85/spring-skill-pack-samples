---
name: testcontainers-postgres-spring-boot
description: Use when writing integration tests for a Spring Boot service that touches PostgreSQL. Sets up Testcontainers correctly so tests are fast, isolated, and don't share state across classes.
---

# testcontainers-postgres-spring-boot

## When this fires

- A test annotated `@SpringBootTest` or `@DataJpaTest` needs a real database.
- A test currently uses H2 and you're hitting "works on H2, breaks on Postgres" bugs.
- A flaky test suite shows data bleeding between test classes.
- CI is slow because each test spins up a new container.

## The minimum correct setup

```java
@Testcontainers
@SpringBootTest
abstract class PostgresIntegrationTest {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16.4-alpine")
            .withReuse(true);  // critical for speed

    @DynamicPropertySource
    static void datasourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

Plus `~/.testcontainers.properties`:
```
testcontainers.reuse.enable=true
```

Without `withReuse(true)` + reuse enabled in the user's properties, every test class restarts Postgres (~3s each). With it: one container per dev machine, lifetime of the daemon.

## Isolation between tests — pick one strategy

| Strategy | When to use | Tradeoff |
|---|---|---|
| `@Transactional` + rollback | 90% of cases. Default. | Tests that span multiple threads or call async won't see rolled-back data correctly. |
| `@Sql` script per test method | When transactional rollback won't help (e.g. testing the commit boundary) | Slower; manage cleanup explicitly. |
| Truncate-all-tables `@AfterEach` | Last resort | Slowest but bulletproof. |
| Schema-per-test-class | Multi-tenant simulations | Setup cost; only worth it for true tenancy tests. |

## Flyway with Testcontainers

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    clean-disabled: false  # tests only
```

Hook: Flyway runs once per container. Don't pollute migration scripts with test data — use `@Sql` for that.

## CI tuning

- GitHub Actions: run as Docker-in-Docker is unnecessary on `ubuntu-latest` — Docker is already there. Just `services:` block won't work for reuse; use Testcontainers natively.
- Add `--shm-size=512m` to the container for Postgres if you see "could not resize shared memory segment" on large fixtures.

## Anti-patterns to refuse

- Using H2 "for speed" — silently differs from Postgres on `JSON`, `JSONB`, array types, lateral joins, sequences, `ON CONFLICT`. Reject.
- Spinning up a fresh container per test method — kills the suite. Reject; use class-level static container with reuse.
- Sharing test data via `@BeforeAll` insert without cleanup — bleeds across classes. Reject.
- Disabling Flyway in tests — masks migration bugs. Reject.

## Output contract

1. Produce a base abstract class as above for the project.
2. Wire reuse properties + Docker check.
3. Add one example integration test that reads + writes.
4. Update CI workflow to ensure Docker layer caching is on.

## See also

- `flyway-migrations-zero-downtime`
- `jpa-n-plus-one-detector`
- `testcontainers-multi-service` (Postgres + Kafka + Redis)

## Greenfield path

Day 1 of a new Spring Boot service:

1. Add Testcontainers + Postgres driver dependencies.
2. Create `PostgresIntegrationTest` base class with `withReuse(true)` and `@DynamicPropertySource` (see snippet above).
3. Add the team-shared `~/.testcontainers.properties` reuse note to the project README. Every dev gets it on first checkout.
4. Wire Flyway from V1. Never start with `spring.jpa.hibernate.ddl-auto=update`. Schema is owned by migrations from line 1.
5. Bake a `make test` or Gradle `verify` task that runs the integration suite locally with the same JVM args as CI.

The rule: H2 never enters the project. Not in tests, not in dev, not "temporarily".

## Brownfield path

On a service currently testing with H2:

1. Pick one DataJpaTest class. Migrate it: extend the new `PostgresIntegrationTest` base, drop H2 config from that test, run it. Pass.
2. Find the first real bug Postgres catches that H2 missed (it will happen — JSON, sequences, `ON CONFLICT`, lateral joins are common). Write it up. Show the team.
3. Migrate the remaining tests in priority order: integration → repository → service. Skip `@WebMvcTest` (no DB) and pure unit tests.
4. Delete the H2 dependency from `build.gradle` / `pom.xml` only after all DataJpaTest classes are migrated AND CI is green for two weeks.
5. Document the migration in an ADR — what broke, how long, what surprised you. (Use `adr-template-and-conventions`.)

What NOT to do on brownfield: bulk-migrate every test in one PR; leave H2 as a "fallback" for "fast local runs" (the runs become 5min on first test because the container starts; reuse fixes that, not H2); skip the ADR.
