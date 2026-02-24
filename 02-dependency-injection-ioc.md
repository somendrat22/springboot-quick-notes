# Module 2: Dependency Injection & IoC Container

---

## 2.1 The Problem — Life Before Dependency Injection

Before understanding DI, let's understand **why it was invented** — what problem was so painful that people created an entire design pattern to solve it.

### Layman Example: Building a House 🏠

Imagine you're building a house. Your house needs **electricity**.

---

#### Scenario A: Without Dependency Injection (You Do Everything Yourself)

```
You're the architect. You draw the house blueprint.
But you also have to:
  1. Go to the power plant
  2. Learn how electricity is generated
  3. Buy transformers, wires, and circuit breakers
  4. Install everything yourself INSIDE your house blueprint

Your house blueprint now contains 200 pages about electricity generation.
```

**Problems:**

| Problem | Explanation |
|---------|-------------|
| **Tight Coupling** | Your house blueprint is glued to one specific power company. If they shut down, your house breaks. |
| **Hard to Change** | Power company switches from coal to solar? You rewrite 200 pages of your blueprint. |
| **Hard to Test** | Want to test if the lights work? You need an actual power plant running. |
| **Cascading Complexity** | The power plant needs water (for cooling). Now your house blueprint also needs to know about water supply to the power plant. |

#### Scenario B: With Dependency Injection (Someone Else Handles It)

```
You're the architect. You draw the house blueprint.
You simply write: "I need a power socket on this wall."

A CONTRACTOR (the IoC Container) reads your blueprint,
goes to the power company, gets the connection,
and delivers electricity to your socket.

You never see the power plant. You don't care if it's coal or solar.
```

**Benefits:**

| Benefit | Explanation |
|---------|-------------|
| **Loose Coupling** | House only knows "I need power." Doesn't care about the source. |
| **Easy to Change** | Switch power provider? Only the contractor's config changes. House blueprint stays the same. |
| **Easy to Test** | Plug in a small battery (mock) to test the lights. No real power plant needed. |
| **No Cascading Complexity** | The house doesn't know about water supply, coal mines, or transformers. |

---

## 2.2 The Problem in Code

### Without DI — Tight Coupling

```java
public class OrderService {

    // OrderService CREATES its own dependency
    private OrderRepository orderRepository = new OrderRepository();
    private EmailService emailService = new EmailService();
    private PaymentGateway paymentGateway = new PaymentGateway();

    public void placeOrder(Order order) {
        orderRepository.save(order);
        paymentGateway.charge(order.getAmount());
        emailService.sendConfirmation(order.getCustomerEmail());
    }
}
```

### What's Wrong Here?

```
Problem 1: TIGHT COUPLING
─────────────────────────
OrderService is GLUED to specific implementations.
Want to switch from MySQL (OrderRepository) to MongoDB (MongoOrderRepository)?
→ You must CHANGE OrderService code.

Problem 2: CASCADING DEPENDENCIES
──────────────────────────────────
What if OrderRepository needs a DataSource?
What if DataSource needs a connection pool?
What if the connection pool needs config?

    OrderService
        → new OrderRepository(
              → new DataSource(
                    → new HikariPool(
                          → new Config("jdbc:mysql://...")
                       )
                 )
           )

OrderService now needs to know about ALL of these!

Problem 3: IMPOSSIBLE TO TEST
─────────────────────────────
Want to unit test OrderService?
- You MUST have a real database running (for OrderRepository)
- You MUST have a real payment gateway (for PaymentGateway)
- You MUST have a real email server (for EmailService)

That's an INTEGRATION test, not a unit test!

Problem 4: SINGLE RESPONSIBILITY VIOLATION
──────────────────────────────────────────
OrderService's job is business logic.
But it's ALSO responsible for creating and managing all its dependencies.
That's two jobs.
```

---

## 2.3 The Solution — Dependency Injection

### The Core Idea

> **Don't create your dependencies. Receive them.**

Instead of `OrderService` creating its own `OrderRepository`, someone **gives** it an `OrderRepository` from outside.

### With DI — Loose Coupling

```java
public class OrderService {

    // Dependencies are DECLARED, not CREATED
    private final OrderRepository orderRepository;
    private final EmailService emailService;
    private final PaymentGateway paymentGateway;

    // Dependencies are INJECTED through the constructor
    public OrderService(OrderRepository orderRepository,
                        EmailService emailService,
                        PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.emailService = emailService;
        this.paymentGateway = paymentGateway;
    }

    public void placeOrder(Order order) {
        orderRepository.save(order);
        paymentGateway.charge(order.getAmount());
        emailService.sendConfirmation(order.getCustomerEmail());
    }
}
```

### What Changed?

| Before (without DI) | After (with DI) |
|---------------------|-----------------|
| `private OrderRepository repo = new OrderRepository();` | `private final OrderRepository repo;` (just a declaration) |
| OrderService **creates** its dependencies | OrderService **receives** its dependencies |
| Coupled to a **specific** implementation | Coupled to an **interface** (abstraction) |
| Hard to test | Easy to test — pass in mocks |

### Testing is Now Easy

```java
@Test
void testPlaceOrder() {
    // Create FAKE (mock) dependencies
    OrderRepository mockRepo = mock(OrderRepository.class);
    EmailService mockEmail = mock(EmailService.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);

    // Inject mocks — no real DB, no real email server, no real payment gateway
    OrderService service = new OrderService(mockRepo, mockEmail, mockPayment);

    Order order = new Order("Laptop", 999.99, "john@example.com");
    service.placeOrder(order);

    // Verify behavior
    verify(mockRepo).save(order);
    verify(mockPayment).charge(999.99);
    verify(mockEmail).sendConfirmation("john@example.com");
}
```

---

## 2.4 Programming to Interfaces (The Power Move)

DI becomes even more powerful when combined with **interfaces**:

```java
// INTERFACE — the contract
public interface OrderRepository {
    void save(Order order);
    Order findById(Long id);
}

// Implementation A — MySQL
@Repository
public class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) { /* MySQL INSERT */ }
    public Order findById(Long id) { /* MySQL SELECT */ }
}

// Implementation B — MongoDB
@Repository
public class MongoOrderRepository implements OrderRepository {
    public void save(Order order) { /* MongoDB insert */ }
    public Order findById(Long id) { /* MongoDB find */ }
}
```

Now `OrderService` depends on the **interface**, not a specific implementation:

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;  // Interface, not a class!

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

Want to switch from MySQL to MongoDB? **Change the config, not the code.**

---

## 2.5 What is IoC (Inversion of Control)?

### Layman Example: Cooking vs. Ordering Food

| Traditional Control | Inverted Control (IoC) |
|--------------------|----------------------|
| **You** decide what to cook, when to cook, how to cook | **The restaurant** decides how to cook — you just order |
| **You** control the entire flow | **Someone else** controls the flow — you just participate |

### In Code

| Traditional | IoC |
|------------|-----|
| **You** create objects: `new OrderService(new OrderRepository())` | **The framework** creates objects and gives them to you |
| **You** control the lifecycle | **The framework** controls the lifecycle |
| Your code calls the framework | **The framework calls your code** |

That last point is the key insight — it's literally **inverting** who's in control.

### The IoC Container in Spring

Spring's IoC container is called the **ApplicationContext**. Think of it as a **factory manager** who:

```
1. SCANS your code at startup
   "Let me find all classes annotated with @Component, @Service, @Repository, @Controller..."

2. CREATES instances (called "beans")
   "OK, I need one OrderController, one OrderService, one OrderRepository..."

3. RESOLVES dependencies
   "OrderController needs OrderService... OrderService needs OrderRepository..."

4. INJECTS dependencies
   "Here you go, OrderController — here's your OrderService with its OrderRepository already wired in."

5. MANAGES lifecycle
   "I'll keep these beans alive as long as the application runs (singleton scope by default)."
```

### How Spring Knows What to Create

```java
@Component    // Generic — "I'm a Spring-managed bean"
@Service      // Semantic — "I'm a service layer bean" (same as @Component, just more descriptive)
@Repository   // Semantic — "I'm a data access bean" (also adds exception translation)
@Controller   // Semantic — "I'm a web controller bean"

// All four are detected during component scanning.
// @Service, @Repository, @Controller are specialized versions of @Component.
```

---

## 2.6 Types of Dependency Injection in Spring

### 1. Constructor Injection (RECOMMENDED)

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final EmailService emailService;

    // Spring auto-detects this constructor and injects the beans
    // (@Autowired is optional if there's only one constructor)
    public OrderService(OrderRepository orderRepository, EmailService emailService) {
        this.orderRepository = orderRepository;
        this.emailService = emailService;
    }
}
```

**Why it's the best:**
- Fields can be `final` → **immutable** (can't accidentally change them)
- All dependencies are **required** — if one is missing, the app won't start (fail fast)
- Easy to **test** — just pass dependencies through the constructor
- No reflection magic — plain Java

### 2. Setter Injection

```java
@Service
public class OrderService {

    private OrderRepository orderRepository;

    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

**When to use:** For **optional** dependencies that have a default value.

### 3. Field Injection (AVOID)

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;  // Injected directly into the field via reflection
}
```

**Why to avoid:**
- Can't make fields `final`
- Hard to test — you need reflection or Spring test context to inject mocks
- Hides dependencies — you can't see them from the constructor
- Easy to add too many dependencies without noticing (leads to God classes)

### Comparison

| Feature | Constructor | Setter | Field |
|---------|------------|--------|-------|
| Immutability (`final`) | Yes | No | No |
| Required dependencies | Yes (fails at startup) | No (can be null at runtime) | No |
| Testability | Easy | Medium | Hard |
| Readability | Clear | OK | Hidden |
| **Recommendation** | **Use this** | Sometimes | Avoid |

---

## 2.7 Bean Scopes

When Spring creates a bean, how many instances does it create?

| Scope | Meaning | Analogy |
|-------|---------|---------|
| `singleton` (default) | ONE instance for the entire application | One kitchen for the whole restaurant |
| `prototype` | NEW instance every time someone asks | Fresh napkin for every customer |
| `request` | ONE instance per HTTP request | One order slip per customer visit |
| `session` | ONE instance per HTTP session | One loyalty card per customer |

```java
@Service
@Scope("singleton")  // default — you don't even need to write this
public class OrderService { }

@Component
@Scope("prototype")  // new instance every time
public class ShoppingCart { }
```

---

## 2.8 Common Interview Questions & Answers

**Q: What is the difference between DI and IoC?**
> **IoC** is the **principle** — "don't call us, we'll call you." The framework controls the flow.
> **DI** is a **technique** to achieve IoC — specifically, injecting dependencies from outside.
> IoC is the WHY. DI is the HOW.

**Q: Why is constructor injection preferred over field injection?**
> Constructor injection allows `final` fields (immutability), makes dependencies explicit, fails fast if a dependency is missing, and is easy to unit test without Spring.

**Q: What is a bean in Spring?**
> A bean is simply an **object that is created, managed, and destroyed by the Spring IoC container**. Any class annotated with `@Component`, `@Service`, `@Repository`, or `@Controller` becomes a bean.

**Q: What happens if two beans implement the same interface?**
> Spring throws `NoUniqueBeanDefinitionException`. You resolve it with:
> - `@Primary` — mark one as the default
> - `@Qualifier("beanName")` — specify which one to inject

```java
@Repository
@Primary
public class MySqlOrderRepository implements OrderRepository { }

@Repository
@Qualifier("mongo")
public class MongoOrderRepository implements OrderRepository { }

// In the service:
@Service
public class OrderService {
    public OrderService(@Qualifier("mongo") OrderRepository repo) {
        // This will inject MongoOrderRepository
    }
}
```

**Q: What is the difference between `@Component` and `@Service`?**
> Functionally, they are **identical** — both register the class as a Spring bean. `@Service` is a semantic alias that signals "this class contains business logic." It makes the code more readable.

---

## 2.9 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| Tight Coupling | Your class creates its own dependencies — hard to change, test, maintain |
| Loose Coupling | Your class receives its dependencies — easy to swap, mock, test |
| Dependency Injection | "Don't create it — receive it" |
| IoC (Inversion of Control) | The framework controls object creation and lifecycle, not you |
| ApplicationContext | Spring's IoC container — the factory that creates and wires all beans |
| `@Component` | "Hey Spring, manage this class as a bean" |
| `@Autowired` | "Hey Spring, inject a bean here" |
| Constructor Injection | Best practice — explicit, immutable, testable |
| `@Primary` | "If there are multiple beans of this type, use me by default" |
| `@Qualifier` | "Inject specifically THIS named bean" |
| Singleton Scope | One instance for the whole app (default) |

---

### Interview Tip 🗣️

> "Dependency Injection means: **Don't create your dependencies — receive them.** The IoC container (Spring's ApplicationContext) acts like a **matchmaker** that knows who needs what and wires everything together at startup. Constructor injection is preferred because it makes dependencies explicit, immutable, and testable."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Client-Server](./01-client-server-architecture.md) | [Next: Controller-Service-Repository ➡️](./03-controller-service-repository.md)
