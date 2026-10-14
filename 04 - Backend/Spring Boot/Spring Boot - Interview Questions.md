---
title: Spring Boot - Interview Questions
aliases: [Spring interview, Spring DI interview, bean scope interview, Spring Boot 面试题]
type: reference
domain: backend
tags: [domain/backend, type/reference, topic/spring-boot, topic/interview, lang/java]
status: draft
created: 2026-10-13
updated: 2026-10-14
version_checked: "Spring Boot 4.1 · Spring Framework 7 — 2026-10"
parent: "[[Spring Boot]]"
related: ["[[Java]]", "[[Kotlin]]", "[[Design Patterns - Interview Questions]]", "[[Backend Interview Questions]]"]
---

# Spring Boot - Interview Questions

> [!info] Reference for [[Spring Boot]] · index: [[Backend Interview Questions]]

> [!abstract] TL;DR
> Spring container questions: **dependency injection** (Q1: constructor injection, multiple beans, profiles, proxies, circular deps, testing with `@MockitoBean`) and **singleton vs prototype scope** (Q2: lifecycle, thread safety, the prototype-into-singleton trap and its three fixes).

## Q1. How do you implement dependency injection in Spring Boot?

**Grading**: IoC container basics, **constructor injection and why**, resolving multiple beans, scopes and proxies, testing.

1. **The container**: Spring's `ApplicationContext` is the IoC container. Beans come from **component scanning** (`@Component`, `@Service`, `@Repository`, `@Controller` under the `@SpringBootApplication` package) and from `@Bean` methods in `@Configuration` classes (for third-party classes you can't annotate). Auto-configuration registers defaults guarded by `@ConditionalOnMissingBean`, so your bean wins.
2. **Constructor injection** (the recommended form): `final` fields, immutable, dependencies visible in the signature, works in plain unit tests with `new`. With a single constructor, `@Autowired` is optional (since Spring 4.3).
3. **Several implementations of one interface**: `@Primary`, `@Qualifier("name")`, or inject `List<T>`/`Map<String, T>` and build a Strategy registry.
4. **Environment-driven wiring**: `@Profile("prod")`, `@ConditionalOnProperty(name = "payments.provider", havingValue = "stripe")`, typed `@ConfigurationProperties`.

```java
public interface PaymentGateway {
    String id();
    Receipt charge(Money amount, String idempotencyKey);
}

@Service
class StripeGateway implements PaymentGateway { /* ... */ }

@Service
class FpxGateway implements PaymentGateway { /* ... */ }

@Service
public class CheckoutService {
    private final Map<String, PaymentGateway> gateways;
    private final OrderRepository orders;

    public CheckoutService(List<PaymentGateway> all, OrderRepository orders) {
        this.gateways = all.stream().collect(Collectors.toUnmodifiableMap(PaymentGateway::id, g -> g));
        this.orders = orders;
    }

    @Transactional
    public Receipt checkout(UUID orderId, String gatewayId) {
        var order = orders.findById(orderId).orElseThrow();
        return gateways.get(gatewayId).charge(order.total(), order.idempotencyKey());
    }
}

@Configuration
class HttpClientConfig {
    @Bean
    RestClient paymentsClient(RestClient.Builder builder, PaymentsProperties props) {
        return builder.baseUrl(props.baseUrl()).build();
    }
}
```

| Detail | What to say |
|---|---|
| Field injection (`@Autowired` on a field) | Works, but no `final`, hidden dependencies, needs reflection or the container in tests. Avoid outside test classes |
| Scopes | Default **singleton** (one per context, so it must be stateless or thread-safe). `prototype`, `request`, `session`. Injecting a prototype into a singleton gives *one* instance; use `ObjectProvider<T>` or a scoped proxy (full answer: Q2) |
| Proxies | `@Transactional`, `@Async`, `@Cacheable` work through AOP proxies. **Self-invocation** (`this.method()`) bypasses the proxy, so the annotation silently does nothing |
| Circular dependencies | Prohibited by default since Spring Boot 2.6 (`spring.main.allow-circular-references`). Fix the design (extract a third bean, or use events) rather than flipping the flag |
| Testing | Unit: `new CheckoutService(List.of(fake), repo)`. Slice/integration: `@SpringBootTest`/`@WebMvcTest` with `@MockitoBean` (the old `@MockBean` was deprecated in Boot 3.4 and removed in 4.0) |
| Kotlin | Primary-constructor injection: `class OrderController(private val orders: OrderService)` |

- **Trap**: "DI = `@Autowired`". DI is the principle (dependencies supplied from outside, depend on abstractions). `@Autowired` is just one way Spring triggers it.
- Deep dive: [[Spring Boot]] · [[Java]] · [[Design Patterns]]

## Q2. What is the difference between singleton and prototype beans in Spring?

**Grading**: lifecycle and thread-safety implications, and the **prototype-into-singleton** trap.

| | Singleton (default) | Prototype |
|---|---|---|
| Instances | One per `ApplicationContext`, cached and shared | New instance per injection point / `getBean()` |
| Created | Eagerly at startup, so wiring errors fail fast (unless `@Lazy`) | Lazily on request, so errors surface at runtime |
| Lifecycle | Full: init callbacks **and** `@PreDestroy` on shutdown | Spring creates and initializes, then forgets it. **`@PreDestroy` never runs**; the caller releases resources |
| Thread safety | Shared across threads, so it must be stateless or thread-safe | Per-use state is fine (not shared) |
| Use for | Services, repositories, clients, config | Stateful or non-thread-safe helpers: builders, per-job context, stateful parsers |
| Declare | Nothing | `@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)` |

**Prototype injected into a singleton** is resolved **once**, at the singleton's creation, so it behaves like a singleton. Ask the container each time instead:

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class ReportBuilder {
    private final List<Row> rows = new ArrayList<>();
    ReportBuilder add(Row r) { rows.add(r); return this; }
    Report build() { return new Report(List.copyOf(rows)); }
}

@Service
class ReportService {
    private final ObjectProvider<ReportBuilder> builders;

    ReportService(ObjectProvider<ReportBuilder> builders) {
        this.builders = builders;
    }

    Report generate(List<Row> data) {
        var builder = builders.getObject();
        data.forEach(builder::add);
        return builder.build();
    }
}
```

| Fix | Notes |
|---|---|
| **`ObjectProvider<T>.getObject()`** | Preferred: explicit, testable, no proxy magic |
| `@Lookup` method | Spring overrides the method via a CGLIB subclass to return a fresh bean |
| Scoped proxy (`proxyMode = ScopedProxyMode.TARGET_CLASS`) | With prototype, **every method call** hits a new instance, so state is lost between calls. Fine for `request`/`session` scopes, surprising for prototype |

- Web-only scopes: `request`, `session`, `application`, `websocket`. Inject them into singletons through a scoped proxy.
- **Traps**: "Spring singleton = GoF Singleton" (it's one per *container*, not per JVM: two contexts mean two instances, and `new` still works). Mutable fields in a singleton (counters, current user) cause races under concurrent requests. Prototype beans holding connections or file handles leak because nothing calls `@PreDestroy`.
- Deep dive: [[Spring Boot]] · Q1

## Related
- [[Spring Boot]] — framework overview
- [[Java]] · [[Kotlin]] — languages
- [[Design Patterns - Interview Questions]] — DI as dependency inversion, Strategy via `Map<String, T>`
- [[Backend Interview Questions]] — interview index

## References
- Spring Framework, dependency injection: https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html
- Spring bean scopes: https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html
- Spring Boot reference: https://docs.spring.io/spring-boot/reference/
