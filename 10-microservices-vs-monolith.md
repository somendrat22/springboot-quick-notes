# Module 10: Microservices vs Monolith

---

## 10.1 What is a Monolith?

A monolith is a **single application** where ALL features, modules, and code live together in **one deployable unit**.

### Layman Example: A Swiss Army Knife 🔪

```
A Swiss Army Knife has:
  - Knife
  - Scissors
  - Screwdriver
  - Bottle opener
  - Corkscrew
  - Tweezers

ALL tools are in ONE device.

If the scissors break → you send the ENTIRE knife for repair.
If you want a bigger screwdriver → you replace the ENTIRE knife.
If 100 people need scissors → you buy 100 entire knives (even though they only need scissors).
```

### In Code — A Monolithic Spring Boot App

```
┌─────────────────────────────────────────────┐
│              MY-SHOP APPLICATION             │
│                                              │
│  ┌──────────────┐  ┌──────────────┐         │
│  │ User Module   │  │ Order Module │         │
│  │ - register    │  │ - placeOrder │         │
│  │ - login       │  │ - getOrders  │         │
│  │ - getProfile  │  │ - cancelOrder│         │
│  └──────────────┘  └──────────────┘         │
│                                              │
│  ┌──────────────┐  ┌──────────────┐         │
│  │ Payment Module│  │ Notification │         │
│  │ - charge      │  │ - sendEmail  │         │
│  │ - refund      │  │ - sendSMS    │         │
│  └──────────────┘  └──────────────┘         │
│                                              │
│  ┌──────────────┐                           │
│  │ Inventory     │                           │
│  │ - checkStock  │                           │
│  │ - updateStock │                           │
│  └──────────────┘                           │
│                                              │
│         ONE database (shared)                │
│         ONE deployment (single JAR/WAR)      │
│         ONE codebase                         │
└─────────────────────────────────────────────┘
```

---

## 10.2 What are Microservices?

Microservices is an architecture where the application is split into **small, independent services**, each responsible for **one business capability**, with its **own database** and **own deployment**.

### Layman Example: A Food Court 🍕🍣🍔🌮

```
Instead of one restaurant that serves EVERYTHING (monolith),
you have a FOOD COURT where:

  🍔 Burger Joint     → only makes burgers (Order Service)
  🍕 Pizza Place      → only makes pizza (Payment Service)
  🍣 Sushi Bar        → only makes sushi (Inventory Service)
  🌮 Taco Stand       → only makes tacos (Notification Service)
  🎫 Central Kiosk    → directs customers (API Gateway)

Each shop:
  - Has its OWN kitchen (own database)
  - Has its OWN staff (own deployment)
  - Can be OPEN or CLOSED independently
  - Can be SCALED independently (3 burger joints, 1 sushi bar)
  - Communicates through a SHARED ordering system (HTTP / message queue)
```

### In Architecture

```
                    ┌──────────────┐
  Client ──────→   │ API GATEWAY  │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐
   │ User        │  │ Order       │  │ Payment     │
   │ Service     │  │ Service     │  │ Service     │
   │ (Port 8081) │  │ (Port 8082) │  │ (Port 8083) │
   └──────┬─────┘  └──────┬─────┘  └──────┬─────┘
          │                │                │
          ▼                ▼                ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐
   │ User DB     │  │ Order DB    │  │ Payment DB  │
   │ (PostgreSQL)│  │ (MySQL)     │  │ (MongoDB)   │
   └────────────┘  └────────────┘  └────────────┘

   + ┌────────────┐  ┌────────────┐
     │ Notification│  │ Inventory   │
     │ Service     │  │ Service     │
     │ (Port 8084) │  │ (Port 8085) │
     └─────────────┘  └─────────────┘
```

---

## 10.3 Monolith vs Microservices — Detailed Comparison

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | One big JAR/WAR — deploy everything at once | Each service deployed independently |
| **Codebase** | One repository, one project | Multiple repos or one monorepo |
| **Database** | Shared database for all modules | Each service has its own database |
| **Scaling** | Scale the ENTIRE app (even if only one module is heavy) | Scale individual services (only the busy ones) |
| **Technology** | One tech stack for everything | Each service can use different languages/databases |
| **Team Structure** | One big team | Small teams own individual services |
| **Failure Impact** | One bug can crash the ENTIRE app | One service fails, others keep running (if designed well) |
| **Communication** | Direct method calls (fast, simple) | HTTP/gRPC/Message queues (network overhead) |
| **Complexity** | Simple to develop, build, and debug | Complex: service discovery, distributed tracing, eventual consistency |
| **Testing** | Easy — everything is in one place | Hard — need integration tests across services |
| **Startup Time** | Can be slow (large app) | Each service starts fast |
| **Data Consistency** | Easy — one DB, use transactions | Hard — distributed transactions, eventual consistency |

---

## 10.4 When to Use What?

### Layman Analogy: Restaurant Growth 📈

```
STAGE 1: You're just starting out
─────────────────────────────────
  → Open a SMALL RESTAURANT (Monolith)
  → One kitchen, one menu, one team
  → Simple, cheap, fast to set up
  → Perfect when you have < 5 cooks

STAGE 2: You're growing
────────────────────────
  → The kitchen is getting crowded
  → Orders are slow because everyone shares one stove
  → The dessert chef accidentally breaks the main oven
  → Consider splitting into a FOOD COURT (Microservices)

STAGE 3: You're Netflix
───────────────────────
  → Thousands of "restaurants" (services)
  → Each one scales independently
  → Different teams own different services
  → You NEED microservices at this scale
```

### Decision Matrix

| Situation | Recommendation |
|-----------|---------------|
| New startup / small team (< 10 devs) | **Monolith** — get to market fast |
| MVP / prototype | **Monolith** — don't over-engineer |
| Well-understood domain, clear boundaries | **Microservices** might work |
| Large team (50+ devs) | **Microservices** — avoid stepping on each other |
| Need to scale specific features independently | **Microservices** |
| Need different tech stacks for different features | **Microservices** |
| Simple CRUD app | **Monolith** — microservices would be overkill |

### The Golden Rule

> **Start with a monolith. Split into microservices when the pain of the monolith exceeds the pain of managing microservices.**

---

## 10.5 How Microservices Communicate

### Synchronous Communication (Request-Response)

```
Order Service needs user info:

  Order Service ──HTTP GET──→ User Service
  Order Service ←──JSON──── User Service

  Like calling someone on the phone — you wait for a response.
```

**Tools:** REST (HTTP), gRPC, Feign Client (Spring)

```java
// Using RestTemplate
@Service
public class OrderService {

    private final RestTemplate restTemplate;

    public OrderService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public UserDTO getUser(Long userId) {
        return restTemplate.getForObject(
                "http://user-service/api/users/" + userId,
                UserDTO.class);
    }
}

// Using Feign Client (declarative — cleaner)
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/api/users/{id}")
    UserDTO getUserById(@PathVariable Long id);
}
```

### Asynchronous Communication (Event-Driven)

```
Order Service places an order:

  Order Service ──publishes event──→ Message Queue (Kafka/RabbitMQ)
                                         │
                         ┌───────────────┼───────────────┐
                         ▼               ▼               ▼
                   Payment Service  Inventory Service  Notification Service
                   (subscribes)     (subscribes)       (subscribes)

  Like sending a letter — you don't wait for a reply.
```

**Tools:** Kafka, RabbitMQ, Amazon SQS

```java
// Producer (Order Service)
@Service
public class OrderService {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public Order placeOrder(Order order) {
        Order saved = orderRepository.save(order);

        // Publish event — don't wait for response
        kafkaTemplate.send("order-events", new OrderEvent(saved.getId(), "ORDER_PLACED"));

        return saved;
    }
}

// Consumer (Notification Service)
@Service
public class NotificationListener {

    @KafkaListener(topics = "order-events", groupId = "notification-group")
    public void handleOrderEvent(OrderEvent event) {
        if ("ORDER_PLACED".equals(event.getType())) {
            emailService.sendOrderConfirmation(event.getOrderId());
        }
    }
}
```

### When to Use Which?

| Communication | When to Use | Example |
|--------------|-------------|---------|
| **Synchronous (REST)** | Need immediate response, simple request-response | Get user details, validate payment |
| **Asynchronous (Events)** | Fire-and-forget, long-running tasks, decoupling | Send email, update analytics, process reports |

---

## 10.6 Key Microservices Patterns

### API Gateway

```
Problem: Client needs to know the address of every service.
Solution: ONE entry point that routes requests to the correct service.

Client → API Gateway → routes to correct service

Tools: Spring Cloud Gateway, Kong, Nginx
```

### Service Discovery

```
Problem: Services run on dynamic ports/IPs (especially in containers).
Solution: A registry where services register themselves.

Order Service starts → registers with Eureka: "I'm at 192.168.1.5:8082"
Payment Service needs Order Service → asks Eureka → gets the address

Tools: Netflix Eureka, Consul, Kubernetes DNS
```

### Circuit Breaker

```
Problem: If Payment Service is down, Order Service keeps calling it and hangs.
Solution: After N failures, STOP calling and return a fallback immediately.

Layman Analogy: A fuse in your house 🔌
  - Too much current → fuse BREAKS → cuts off electricity → prevents fire
  - Too many failures → circuit OPENS → stops calling → prevents cascade failure

States:
  CLOSED → normal operation, requests go through
  OPEN   → too many failures, return fallback immediately
  HALF-OPEN → try one request to see if service is back

Tools: Resilience4j (modern), Hystrix (deprecated)
```

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return restTemplate.postForObject("http://payment-service/pay", request, PaymentResponse.class);
    }

    // Fallback — runs when circuit is open
    public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
        return new PaymentResponse("PENDING", "Payment service is temporarily unavailable");
    }
}
```

### Distributed Tracing

```
Problem: A request touches 5 services. Where did it slow down? Where did it fail?
Solution: Assign a unique TRACE ID to each request and track it across all services.

Request: GET /api/orders/42
  Trace ID: abc-123
    → API Gateway (50ms)          [abc-123]
    → Order Service (120ms)       [abc-123]
    → User Service (30ms)         [abc-123]
    → Payment Service (800ms) 🐌  [abc-123]  ← Found the bottleneck!
    → Notification Service (20ms) [abc-123]

Tools: Zipkin, Jaeger, Spring Cloud Sleuth + Micrometer
```

---

## 10.7 The Challenges of Microservices

| Challenge | What It Means | Solution |
|-----------|-------------|----------|
| **Network Latency** | Every service call goes over the network (slower than in-memory) | Minimize calls, use caching, async communication |
| **Data Consistency** | No shared DB → can't use simple transactions across services | Saga pattern, eventual consistency, event sourcing |
| **Distributed Debugging** | Bugs span multiple services — hard to trace | Distributed tracing (Zipkin), centralized logging (ELK) |
| **Deployment Complexity** | 20 services = 20 deployments, 20 configs, 20 health checks | Docker, Kubernetes, CI/CD pipelines |
| **Service Discovery** | How do services find each other? | Eureka, Consul, K8s DNS |
| **Cascading Failures** | One slow service blocks everything | Circuit breakers, timeouts, bulkheads |
| **Testing** | Integration testing across services is hard | Contract testing (Pact), consumer-driven contracts |

---

## 10.8 The Modular Monolith — The Middle Ground

If microservices feel too complex but your monolith is getting messy, consider a **modular monolith**:

```
┌─────────────────────────────────────────────┐
│            MODULAR MONOLITH                  │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ User      │  │ Order     │  │ Payment   │  │
│  │ Module    │  │ Module    │  │ Module    │  │
│  │           │  │           │  │           │  │
│  │ Controller│  │ Controller│  │ Controller│  │
│  │ Service   │  │ Service   │  │ Service   │  │
│  │ Repository│  │ Repository│  │ Repository│  │
│  └──────────┘  └──────────┘  └──────────┘  │
│       │              │              │        │
│       ▼              ▼              ▼        │
│  ┌─────────────────────────────────────┐    │
│  │         SHARED DATABASE              │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ONE deployment, but CLEAR module boundaries │
│  Modules communicate through INTERFACES,     │
│  not direct DB queries across modules.       │
└─────────────────────────────────────────────┘
```

**Benefits:**
- Simple deployment (still one JAR)
- Clear boundaries (easy to split into microservices later)
- No network overhead (still in-memory calls)
- Transaction safety (shared DB, can use @Transactional)

---

## 10.9 Common Interview Questions & Answers

**Q: What are microservices?**
> Microservices is an architectural style where an application is composed of small, independent services that each handle one business capability, have their own database, and communicate over the network (HTTP or messaging).

**Q: When would you choose a monolith over microservices?**
> For small teams, new products, MVPs, or simple applications. The overhead of managing microservices (deployment, networking, distributed data) is not justified until the monolith becomes a bottleneck in terms of team productivity or scaling.

**Q: What is the biggest challenge with microservices?**
> **Data consistency.** Without a shared database, you can't use simple ACID transactions across services. You need patterns like Sagas (choreography or orchestration) and accept eventual consistency.

**Q: How do microservices communicate?**
> **Synchronously** via REST/HTTP or gRPC (for request-response). **Asynchronously** via message brokers like Kafka or RabbitMQ (for events). Async is preferred for decoupling and resilience.

**Q: What is an API Gateway?**
> A single entry point for all clients. It routes requests to the correct microservice, handles cross-cutting concerns like authentication, rate limiting, logging, and load balancing. Examples: Spring Cloud Gateway, Kong, Nginx.

**Q: What is a Circuit Breaker?**
> A pattern that prevents cascade failures. If a downstream service is failing, the circuit breaker "opens" and returns a fallback response immediately instead of waiting and timing out. After a cooldown period, it tries again ("half-open" state).

**Q: What is the Saga pattern?**
> A way to manage distributed transactions across microservices. Instead of one big ACID transaction, you have a sequence of local transactions. If one fails, compensating transactions undo the previous steps.
> **Choreography:** Each service publishes events that trigger the next step.
> **Orchestration:** A central orchestrator tells each service what to do.

**Q: Can each microservice use a different programming language?**
> Yes — that's one of the benefits. A payment service could be in Java, a notification service in Python, and an analytics service in Go. They communicate via language-agnostic protocols (HTTP/REST, gRPC, message queues).

---

## 10.10 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| Monolith | One codebase, one deployment, one database — simple but doesn't scale well |
| Microservices | Many small services, each with own DB and deployment — scalable but complex |
| API Gateway | Single entry point that routes requests to the correct service |
| Service Discovery | Registry where services register and find each other (Eureka) |
| Circuit Breaker | Stop calling a failing service, return fallback instead (Resilience4j) |
| Sync Communication | REST/HTTP — wait for response (like a phone call) |
| Async Communication | Message queue — fire and forget (like sending a letter) |
| Saga Pattern | Distributed transactions via a chain of local transactions + compensations |
| Modular Monolith | One deployment with clear internal module boundaries — best of both worlds |
| When to use Monolith | Small team, new product, MVP, simple app |
| When to use Microservices | Large team, need independent scaling, different tech stacks |

---

### Interview Tip 🗣️

> "I always recommend **starting with a well-structured monolith** and extracting microservices only when there's a clear need — like independent scaling, team autonomy, or different tech requirements. Microservices solve organizational and scaling problems but introduce complexity in data consistency, debugging, and deployment. The middle ground is a **modular monolith** — one deployment with clear module boundaries that can be split later."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Spring Security](./09-spring-security-basics.md)
