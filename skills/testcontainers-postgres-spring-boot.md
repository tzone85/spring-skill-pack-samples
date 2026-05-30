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
