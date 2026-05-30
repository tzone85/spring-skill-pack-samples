---
name: jpa-n-plus-one-detector
description: Use when reviewing Spring Data JPA code, after adding a new endpoint that loads related entities, or when investigating slow list endpoints. Detects classic N+1 patterns and recommends fetch joins, entity graphs, or projections.
---

# jpa-n-plus-one-detector

## When this fires

- A repository method returns a list and the entity has lazy `@ManyToOne` / `@OneToMany` / `@ManyToMany` relationships.
- A service iterates a list and calls a getter on a lazy relationship inside the loop.
- A controller returns DTOs that include nested children loaded from separate queries.
- A flame graph or APM shows N+1 query bursts on the slow path.

## How to detect

1. Turn on Hibernate statistics in `application-test.yml`:
   ```yaml
   spring:
     jpa:
       properties:
         hibernate:
           generate_statistics: true
   logging:
     level:
       org.hibernate.SQL: DEBUG
       org.hibernate.orm.jdbc.bind: TRACE
   ```
2. Write a Testcontainers integration test that hits the endpoint with at least 10 parent rows and asserts query count using `SessionFactory.getStatistics().getPrepareStatementCount()`. Fail the test if count > expected.
3. Add `assertj-db` or `datasource-proxy` for prod-shape harnesses if Hibernate stats aren't enough.

## How to fix — pick the smallest tool that solves the case

| Symptom | Fix | Tradeoff |
|---|---|---|
| List endpoint loads `@ManyToOne` parents | `JOIN FETCH` in JPQL or `@EntityGraph(attributePaths={"parent"})` on repo method | Cartesian product if you join multiple collections — only join ONE collection |
| List endpoint loads `@OneToMany` children | `@EntityGraph` + pagination via `Slice` not `Page` (count query breaks) | Lose total count |
| Need many collections at once | Two queries: parents + `findChildrenByParentIdIn(ids)`, stitch in service | Manual, but predictable |
| DTO-shaped response | JPQL projection `select new com.x.Dto(...)` or interface projection | Skips entity loading entirely — fastest |
| Reporting / read-heavy view | Dedicated read model (separate `@Entity` mapped to a SQL view) | Migration cost |

## Anti-patterns to refuse

- `@OneToMany(fetch = EAGER)` — fixes N+1 by always paying for it. Reject.
- `Hibernate.initialize()` in the service — works but hides the problem from tests. Reject.
- Cartesian fetch joins on two collections — produces `parents × childrenA × childrenB` rows. Reject; split into two queries.
- `findAll()` with no pagination in any endpoint that's not `/admin`. Reject.

## Output contract

When invoked on a code chunk, the skill should:

1. List every N+1 risk with file:symbol references.
2. For each risk, name the smallest fix from the table.
3. Provide the patched code snippet — not pseudocode.
4. Add a Testcontainers test that fails before the patch and passes after.

## See also

- `jpa-fetch-strategies-n-plus-one` (deep dive)
- `jpa-projections-and-views` (DTO projections)
- `testcontainers-postgres-spring-boot` (test harness)

## Greenfield path

On a new Spring Boot service: bake the n+1 guardrails in from day 1.

1. Generate the Testcontainers integration test base (see `testcontainers-postgres-spring-boot`).
2. Add this rule to your `application-test.yml`: `spring.jpa.properties.hibernate.generate_statistics=true`.
3. In every list-returning repository method's integration test, assert: `assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(expectedQueries)`.
4. Default new repos to interface projections, not entity returns. Only return entities when you know the consumer mutates them.
5. ArchUnit rule: no `@OneToMany(fetch = EAGER)` anywhere in the codebase. Fails the build on commit.

The pattern is: make the smell impossible to ship, not just easy to spot.

## Brownfield path

On a legacy Spring Boot service where n+1s are already in prod:

1. Don't refactor everything. Pick the top 3 slow endpoints (use Actuator `/metrics` + APM heat-map). Quantify the cost.
2. For each, write the integration test that demonstrates the n+1 (use Hibernate statistics). The failing test is the ticket.
3. Apply the smallest fix from the matrix above. Resist the urge to also rename, modularise, or restyle.
4. Add the ArchUnit `no-EAGER` rule as a WARNING (not failure) for a sprint, then flip to failure. Lets the team triage the existing offenders.
5. Track p99 latency on those endpoints before/after. Tell the team. Boring slides win promotions.

What NOT to do on brownfield: rip out `@OneToMany` mappings in bulk; introduce DTO projections everywhere overnight; ship `Hibernate.initialize` calls "as a stopgap". All of these create new bugs and hide the old ones.
