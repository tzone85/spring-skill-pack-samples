---
name: hexagonal-architecture-spring-boot
description: Use when starting a new Spring Boot service, refactoring a tangled service-layer codebase, or onboarding a team to Ports and Adapters. Establishes module boundaries that survive growth — domain stays framework-free.
---

# hexagonal-architecture-spring-boot

## When this fires

- Greenfield service where you control structure.
- Existing service where `@Service` classes import `org.springframework.data.*` AND business logic AND HTTP DTOs — the symptom of no boundary.
- Team is about to swap a major adapter (e.g. Postgres → DynamoDB, REST → gRPC) and dreads it.

## The cut

Domain depends on nothing. Application depends on domain. Adapters depend on application + domain. Spring lives only in adapters and the bootstrap.

```
app-domain/      <- pure Java, no Spring, no JPA, no Jackson
app-application/ <- use cases, port interfaces, no Spring (or Spring Context only)
app-adapter-rest/   <- @RestController, DTOs
app-adapter-jpa/    <- @Entity, repositories
app-adapter-kafka/  <- producers, consumers
app-bootstrap/   <- @SpringBootApplication, wiring, application.yml
```

Multi-module Gradle or Maven. ArchUnit test enforces the dependency direction.

## Domain layer rules

- No annotations from frameworks. No `@Entity`, no `@JsonProperty`, no `@Component`.
- Aggregate roots own their invariants. Setters are private or absent.
- Value objects are records, equals/hashCode by value.
- Domain events as records.
- No nullable returns in public API — use `Optional` or throw a domain exception.

```java
// app-domain
public record OrderId(UUID value) { ... }

public final class Order {
    private final OrderId id;
    private final List<OrderLine> lines;
    private Status status;

    public static Order place(OrderId id, List<OrderLine> lines) {
        if (lines.isEmpty()) throw new EmptyOrderException(id);
        return new Order(id, lines, Status.PLACED);
    }

    public OrderConfirmed confirm() {
        if (status != Status.PLACED) throw new IllegalStateException();
        this.status = Status.CONFIRMED;
        return new OrderConfirmed(id, Instant.now());
    }
}
```

## Application layer — ports

Inbound ports (driving): use case interfaces.
Outbound ports (driven): repository / gateway interfaces.

```java
// app-application
public interface PlaceOrderUseCase {
    OrderId place(PlaceOrderCommand cmd);
}

public interface OrderRepository {              // outbound port
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

@Component
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository orders;
    private final EventPublisher events;
    // constructor injection
    public OrderId place(PlaceOrderCommand cmd) { ... }
}
```

## Adapter layer

```java
// app-adapter-rest
@RestController
@RequestMapping("/api/orders")
class OrderController {
    private final PlaceOrderUseCase useCase;       // depends on the PORT, not the impl

    @PostMapping
    OrderResponse create(@Valid @RequestBody PlaceOrderRequest req) {
        var id = useCase.place(req.toCommand());
        return new OrderResponse(id.value());
    }
}

// app-adapter-jpa
@Repository
class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderRepo repo;
    private final OrderMapper mapper;

    @Override public void save(Order order) { repo.save(mapper.toEntity(order)); }
    @Override public Optional<Order> findById(OrderId id) { ... }
}
```

## Enforcement — ArchUnit

```java
@AnalyzeClasses(packages = "com.example.app")
class ArchitectureTest {
    @ArchTest static final ArchRule domain_depends_on_nothing =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..", "jakarta.persistence..", "com.fasterxml.jackson..");

    @ArchTest static final ArchRule application_depends_only_on_domain_or_spring_context =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "jakarta.persistence..", "com.fasterxml.jackson..", "org.springframework.web..");
}
```

## Anti-patterns to refuse

- "Hexagonal" with one Maven module — the boundary is enforced by package names only, easy to violate, gives no compile-time safety. Reject; multi-module.
- Domain entities that ARE the JPA entities — couples DB schema to invariants. Reject; map at the adapter boundary.
- Reusing JPA entity as REST DTO — leaks DB shape to public contract. Reject; separate DTO.
- One mega `*UseCase` interface with 20 methods — defeats the point. Reject; one interface per use case.

## Output contract

When invoked on a greenfield project: generate the 5 modules, the ArchUnit test, a `Makefile` or Gradle task `verify` that runs all checks, and one end-to-end example use case wired through.

## See also

- `ddd-aggregates-and-repositories`
- `event-driven-spring-events` (in-process events for use-case boundaries)
- `outbox-pattern` (cross-service event delivery)
- `adr-template-and-conventions` (record the choice)
